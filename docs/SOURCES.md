# Data sources

Per-source reference for this pack: what each input collects, the binaries
it calls, and the sourcetype it lands under. Overview, data flow and field
tables stay in [`README.md`](../README.md).

### Apple Unified Logs (`in_apple_unified_logs`) — Native 4.18 Source

- **Type**: `apple_unified_logs` (macOS Edge only)
- **Polling**: 5 second internal poll (Cribl-managed)
- **Predicate**: narrowed (v0.4.0) to the panic/thermal/jetsam signal
  path — kernel jetsam/memorystatus/panic/thermal/IOGPU/AGX events, DumpPanic,
  ReportCrash, thermalmonitord, and the WindowServer watchdog-timeout/GPU-restart
  strings. The prior broad `subsystem BEGINSWITH "com.apple"` predicate ran 1.33M
  events/day (dominated by runningboardd/WallpaperAgent noise) and buried the one line
  that mattered; this predicate measures 63k/day, a 21x reduction, while still catching
  it. Full expression in [`default/inputs.yml`](default/inputs.yml).
- **Read mode**: no `readMode` key is set. `lastEntry` is **not** a valid enum value
  (validator only accepts `oldest`/`newest`) — it silently rejected this input on every
  restart and it had never run once. Do not set it; the default (`newest`) is correct.
- **Sourcetype** (assigned in pipeline): `macos:unified_log`
- **Index**: `os`
- **Replaces**: v0.2 exec sources `macos-windowserver-health` and `macos-jetsam-events`.
  Filtering for those specific signals now lives in Stream pipelines.
- **`memory pressure` is deliberately unscoped**, unlike the other clauses in its group
  — see the comment in [`default/inputs.yml`](default/inputs.yml) for the measurement
  that found the kernel-scoped form matches zero events, versus 73–93 unscoped over the
  same window.
- **Known unverified clause**: `GPU restart` / `gpu hang` have no confirmed macOS
  emitter — kept on a rare-vs-invented judgment call, not evidence.
- **Known gap**: `low swap` is kernel-scoped by assumption only, the same assumption
  already found wrong for `memory pressure`; unverified because no observed window has
  had a nonzero swap rate.
- **Known gap, deliberate**: `Metal` and `SkyLight` are not in the predicate — they would
  add roughly 26% volume from routine compositor chatter, and the compositor-starvation
  signal that matters is already caught by `DumpPanic` and `userspace watchdog timeout`.

### System Metrics (`in_system_metrics`) — Native 4.18 Source

- **Type**: `system_metrics` (Linux + macOS Edge)
- **Polling**: 60 seconds
- **Collectors**: CPU, memory, disk, network, system, process (configure detail level
  per category in Cribl UI). Container + GPU collectors stay disabled on macOS.
- **Sourcetype** (assigned in pipeline): `macos:system:metrics`
- **Index**: `mac_perf`
- **Replaces**: v0.2 exec sources `macos-memory-pressure`, `macos-vm-stat`,
  `macos-disk-io`, `macos-process-top`.

### Thermal Status (`macos-thermal`) — Exec Source

- **Interval**: 60 seconds
- **Command**: `pmset -g therm`
- **Sourcetype**: `macos:system:thermal`
- **Captures**: Thermal warning level, performance warning level, CPU power status
- **Why exec**: System Metrics Source on macOS doesn't yet surface system-wide thermal
  pressure / CPU power throttling state.
- **Requires**: No special privileges

### Power Metrics (`macos-power-metrics`) — Exec Source

- **Interval**: 300 seconds (5 minutes)
- **Command**: `powermetrics --samplers tasks,battery,cpu_power,gpu_power,ane_power,thermal ...`
  piped through `plutil -replace timestamp -string "" -o - -` then `plutil -convert json`
  — full command in [`default/inputs.yml`](default/inputs.yml). The `-replace` step is
  required: `plutil -convert json` cannot represent the plist `<date>` type that
  powermetrics always emits in its per-sample header, so every sample failed to convert
  before this fix (v0.4.0). `-remove` was tried first but hard-fails whenever the
  key is briefly absent; `-replace` creates it and cannot fail that way.
- **Sourcetype**: `macos:perf:powermetrics`
- **Index**: `mac_perf`
- **Captures**: Per-process energy impact (top 10), CPU package power (mW), GPU power,
  ANE power, thermal pressure state, processor frequency
- **Anomaly**: `thermal.thermal_pressure !== 'Nominal'` sets `anomaly=true`
- **Why exec**: No native 4.18 Source exposes per-process energy / ANE power.
- **Requires**: Root privileges

### Battery Health (`macos-power-battery`) — Exec Source

- **Interval**: 60 seconds
- **Command**: Bash combining `pmset -g batt` + `ioreg -r -c AppleSmartBattery`
- **Sourcetype**: `macos:power:battery`
- **Captures**: Charge percentage, power source (AC/Battery), charging state,
  cycle count, max/design/current capacity, temperature, voltage
- **Derived**: `battery_health_percent = max_capacity / design_capacity × 100`
- **Anomaly**: `battery_health_percent < 80` sets `anomaly=true`
- **Why exec**: No native Source exposes battery cycle count / design capacity /
  IOKit `AppleSmartBattery` properties.
- **Requires**: No special privileges

### Wired Memory (`macos-wired-memory`) — Exec Source

- **Interval**: 60 seconds
- **Command**: chained stock binaries (`vm_stat`, `sysctl`, `ps`, `awk`) — no script file;
  full command in [`default/inputs.yml`](default/inputs.yml)
- **Sourcetype**: `macos:perf:wired_memory`
- **Index**: `mac_perf`
- **Captures**: `wired_bytes` (from `vm_stat` "Pages wired down" × `sysctl hw.pagesize`,
  never hardcoded), `wired_ceiling_mb`/`wired_ceiling_bytes` (`sysctl iogpu.wired_limit_mb`),
  `wired_ratio`, and `resident_bytes` (summed across all processes via `ps`, so leaked
  wired memory — `wired - resident - baseline` — is directly calculable)
- **Why**: Wired memory vs. the `iogpu.wired_limit_mb` ceiling predicts this failure
  class — a wired/ceiling ratio above ~0.9 has preceded a crash in observed data, and
  nothing tracked that ratio before this input. Neither wired memory nor the ceiling is
  a native `system_metrics` field (checked 4.19 docs; only `memory_percent` is exposed).
- **Requires**: No special privileges
- **Known follow-up**: `macos:perf:wired_memory` is a new stream and has no matching
  stanza yet in `VisiCore_TA_AI_Observability`'s `props.conf` — this data ships unparsed
  by that TA until a stanza is added there (a separate repo/change, not done here).

### Crash Reports (`in_macos_crashreports_sys`, `in_macos_crashreports_user`) — File Sources

- **Interval**: 60 seconds (file poll), `tailOnly: true` (new reports only)
- **Paths**: `/Library/Logs/DiagnosticReports/` and `$HOME/Library/Logs/DiagnosticReports/`
- **Filenames**: `*.ips`, `*.panic`, `*.crash`, `*.diag`, `*.hang`, `*.spin`,
  `*.shutdownStall` — `.spin` and `.shutdownStall` added in v0.4.0; `.spin` carries
  the blocked-thread stack for a hung process (the single most diagnostic artifact for a
  blocked main-thread hang) and `.shutdownStall` marks an unclean shutdown.
- **Sourcetype**: `macos:crashreport`
- **Index**: `os`
- **Event breaker**: `MacOS Crash Reports` (see [`default/breakers.yml`](default/breakers.yml)) —
  **one event per file**, not per line. The default newline-delimited breaker shattered a
  single WindowServer `.ips` into 2,560 separate events, scattering the `ws_main_thread`
  stack across unreadable rows and stamping `_time` on only the one line that carried a
  timestamp. The custom `eventBreakerRegex` fires only on a report's opening line: an
  object whose first key's value is a quoted timestamp string (`{"timestamp":"..."`), or
  `Date/Time:`/`Use spindump` for `.spin`/`.shutdownStall`. A plain `{"` anchor is not
  enough — some `.diag` bodies (the `SFA-*` family) are compact single-line JSON starting
  at column 0, so `{"` alone matches the header *and* the body and splits the file in
  two, one half stamped at arrival time. Requiring the quoted `"timestamp":"` value
  excludes those bodies structurally, since their embedded timestamp (when present) is a
  bare epoch float, not a quoted string — verified against every crash-report artifact
  swept by these globs on this host (`/Library` + `$HOME` DiagnosticReports, recursive,
  Retired/ included), one event each, zero exceptions. `maxEventBytes` is 16MiB — the
  largest observed `.spin` is already 3.6MB against the prior 4MB ceiling, and `.spin`
  size tracks live thread count.
- **`crash_time_in_band`** (boolean, every event): true when the event's own bytes carry
  a report timestamp the breaker could parse; false for `.shutdownStall`, which has no
  timestamp anywhere in the breaker's scan window and always falls back to arrival time.
  Never writes `_time` itself — the breaker remains the sole `_time` writer — and is a
  claim about the bytes, not about provenance, so it stays true regardless of whether the
  breaker's own parse happened to succeed.
- **Requires**: Read access to `/Library/Logs/DiagnosticReports/` (root-owned directory,
  world-readable listing; per-file permissions vary)
- **Known follow-up**: `macos:crashreport` has no matching stanza in
  `VisiCore_TA_AI_Observability`'s `props.conf` — this data ships unparsed by that TA
  until a stanza is added there (a separate repo/change, not done here). Higher priority
  than the `macos:perf:wired_memory` gap below: this stream already has real event
  volume.
