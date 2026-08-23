# cc-edge-the-mac-pack-io

Cribl Edge Pack for comprehensive macOS system, power, and performance monitoring.

> Supersedes `cc-edge-macos-system` and `cc-edge-macos-power` — both repos are archived.

## Overview

Collects macOS telemetry on a Cribl Edge Node running natively on macOS. As of v0.3.0
the pack uses Cribl 4.18's native macOS Sources for broad host-metrics and Unified-Log
capture, falling back to Exec sources only for data the native Sources don't yet expose
(per-process energy via `powermetrics`, battery cycle/health via `ioreg`, and macOS
thermal pressure via `pmset`).

Design rule: **Edge captures broadly, Stream filters — unless volume makes that
uneconomical.** `in_apple_unified_logs`'s predicate is the one exception (v0.4.0):
`subsystem BEGINSWITH "com.apple"` ran 1.33M events/day, most of it noise, and that cost
was itself why the one signal that mattered got lost. New use-case filtering still
belongs in Stream; this predicate stays narrow at the Edge on purpose.

Events flow through Cribl Stream into Splunk (or any destination).

This pack is the macOS equivalent of Splunk's Splunk_TA_nix and Splunk_TA_windows —
delivering comprehensive operating system telemetry via Cribl Edge instead of scripted
inputs on a forwarder.

## Origin Story

This pack was born from a real incident: a 60-second MacBook UI freeze went undetected
because WindowServer ping timeout events required manual log forensics to discover.

In v0.1–v0.2 the pack ran a tightly-filtered `log show ... WindowServer ... ping` exec
loop. In v0.3 that filtering moves to Stream — the Edge now captures the full Apple
Unified Log stream broadly, and Stream pipelines extract the ping-timeout signal (and
any other subsystem-of-interest) downstream.

## Data Sources

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

## Data Flow

```text
Apple Unified Logging (narrowed predicate)                Cribl native
   jetsam/panic/thermal/watchdog signal path ┐              4.18 Source
macOS kernel/system metrics                  ┤
   CPU, memory, disk, network, processes     ┘
pmset / powermetrics / ioreg / vm_stat       ┐              Cribl Exec
DiagnosticReports crash files                ┤              + File
                                             ┘              Sources
                       │
                       ▼
                 Cribl Edge (native macOS, /opt/cribl/)
                       │ TLS
                       ▼
                 Cribl Stream (per-use-case routing/filtering happens here)
                       │ HEC
                       ▼
                 Splunk:
                   index=os         sourcetype=macos:unified_log | macos:system:thermal
                                    | macos:power:battery | macos:crashreport
                   index=mac_perf   sourcetype=macos:system:metrics | macos:perf:powermetrics
                                    | macos:perf:wired_memory
```

## Anomaly Detection

Edge keeps anomaly detection ONLY for events that come from the exec sources
(powermetrics, battery). Anomaly detection for memory pressure, WindowServer ping
timeouts, and Jetsam events moves to Stream — the broad Apple Unified Logs / System
Metrics streams already carry the underlying data; Stream pipelines extract the signal.

| Condition | anomaly_reason | Detected at |
| ----------- | ---------------- | ------------- |
| Battery health < 80% | `battery_health_degraded` | Edge (this pack) |
| Thermal pressure not Nominal | `thermal_pressure_elevated` | Edge (this pack) |
| Memory pressure critical | (Stream-side, future) | Stream |
| WindowServer ping timeout | (Stream-side, future) | Stream |
| Jetsam event | (Stream-side, future) | Stream |

Use these fields in Splunk to alert on what Edge does flag:

```spl
(index=os OR index=mac_perf) anomaly=true
| stats count by host, anomaly_reason, _time
```

## Usage

| Sourcetype namespace | Index | Source type |
| --- | --- | --- |
| `macos:unified_log` | `os` | Native (apple_unified_logs) |
| `macos:system:metrics` | `mac_perf` | Native (system_metrics) |
| `macos:system:thermal` | `os` | Exec |
| `macos:power:battery` | `os` | Exec |
| `macos:perf:powermetrics` | `mac_perf` | Exec |
| `macos:perf:wired_memory` | `mac_perf` | Exec |
| `macos:crashreport` | `os` | File |

The `sourcetype` eval in `default/pipelines/main/conf.yml` is an explicit per-datatype map,
not a fallback chain — an unmatched `datatype` is left with `sourcetype` undefined rather
than silently guessed, so a new input that isn't wired into the map is visibly wrong
instead of mislabeled.

To customize, create local overrides in `/opt/cribl/local/cc-edge-the-mac-pack-io/`:

```yaml
# /opt/cribl/local/cc-edge-the-mac-pack-io/inputs.yml
inputs:
  in_apple_unified_logs:
    predicate: 'TRUEPREDICATE'   # capture everything, not just the panic signal path
  in_system_metrics:
    pollingInterval: 30          # increase metric frequency
  macos-power-metrics:
    disabled: true               # disable if root unavailable
  macos-wired-memory:
    interval: 30                 # tighten the panic-predictor sample rate
```

## Installation

1. Copy pack to Cribl Edge or install via API:

   ```bash
   cp cc-edge-the-mac-pack-io.crbl /opt/cribl/state/packs/

   curl -X POST http://localhost:9000/api/v1/packs \
     -H "Content-Type: application/json" \
     -d '{"source":"cc-edge-the-mac-pack-io.crbl"}'
   ```

2. Commit and restart:

   ```bash
   curl -X POST http://localhost:9000/api/v1/version/commit
   curl -X POST http://localhost:9000/api/v1/system/settings/restart
   ```

## Fields

### Power Metrics Events (`macos:perf:powermetrics`)

| Field | Type | Description |
| ------- | ------ | ------------- |
| top_processes | array | Top 10 processes by energy impact `{name, pid, energy_impact}` |
| thermal_pressure | string | Thermal pressure state (`Nominal`, `Moderate`, `Heavy`, `Trapping`) |
| processor | object | CPU package power (mW), frequency per cluster |
| gpu | object | GPU power (mW) |
| ane_power | number | Apple Neural Engine power (mW) |

### Battery Events (`macos:power:battery`)

| Field | Type | Description |
| ------- | ------ | ------------- |
| charge_percent | number | Current charge percentage |
| power_source | string | `AC` or `Battery` |
| charging_state | string | `charging`, `discharging`, `charged`, or `unknown` |
| cycle_count | number | Battery charge cycle count |
| max_capacity | number | Current maximum capacity (mAh) |
| design_capacity | number | Original design capacity (mAh) |
| battery_health_percent | number | Derived: `max_capacity / design_capacity × 100` |
| temperature | number | Battery temperature (hundredths of °C) |
| voltage | number | Battery voltage (mV) |

### Native Source Events

The native `apple_unified_logs` and `system_metrics` Sources emit Cribl-defined event
shapes — see Cribl's [macOS System Metrics Details](https://docs.cribl.io/edge/system-metrics-macos-output)
and [Apple Unified Logs Source](https://docs.cribl.io/edge/sources-apple-unified-logs)
reference docs for the full field listings. Edge passes them through unchanged; Stream
pipelines do downstream extraction.

### Common Fields (all sourcetypes)

| Field | Type | Description |
| ------- | ------ | ------------- |
| index | string | `os` for unified-log + thermal + battery + crash reports; `mac_perf` for system metrics + powermetrics + wired memory |
| sourcetype | string | See namespace table above |
| host | string | Hostname of the Edge node |
| anomaly | boolean | `true` when anomaly detected (exec sources only — see Anomaly Detection) |
| anomaly_reason | string | Anomaly type identifier |

## Release Notes

### v0.4.0 (2026-08-23)

Fixes for several macOS telemetry sources that were silently non-functional, broken,
or incomplete, plus a new source for a metric that nothing tracked before.

- **`in_apple_unified_logs` had never run**: `readMode: lastEntry` is not a valid enum
  value and the input was rejected on every restart. Removed the key (default `newest`
  is correct) and narrowed the predicate to the panic/thermal/jetsam signal path — a
  21x volume reduction (1.33M/day → 63k/day) applied in the same change so fixing the
  enum alone wouldn't flood the destination.
- **`macos-power-metrics` could never convert to JSON**: `plutil -convert json` cannot
  represent the plist `<date>` type that powermetrics always emits; every sample failed.
  Fixed with `plutil -replace timestamp -string "" -o - -` before the JSON conversion.
- **New crash-report globs** `*.spin` and `*.shutdownStall` on the crash-report file
  inputs — `.spin` carries the blocked-thread stack for a hang, `.shutdownStall` marks
  an unclean shutdown; neither was collected before.
- **New event breaker** `MacOS Crash Reports` (`default/breakers.yml`) — one event per
  file, breaking only on a report's opening line (`{"..."` / `Date/Time:` / `Use
  spindump`), since Cribl 4.19 has no true one-event-per-file ruleType. Verified against
  real crash-report samples: a 2,560-line WindowServer `.ips` produces exactly one event
  containing the `ws_main_thread` stack.
- **New `in_macos_crashreports_sys` / `in_macos_crashreports_user` File inputs** — crash
  reports were not collected by this pack at all before this release.
- **New `macos-wired-memory` Exec input** — wired memory vs. the `iogpu.wired_limit_mb`
  ceiling is the direct predictor of this failure class and was tracked nowhere.
- **Sourcetype routing fixed**: the pipeline's sourcetype eval was a fallback chain that
  silently mislabeled any unmatched `datatype`; replaced with an explicit per-datatype
  map with no fallback.
- **Removed** the orphaned `mac-perf-snapshots` File input and its `macos:perf:snapshot`
  sourcetype — the external NDJSON collector that fed it was deliberately deleted.
  **BREAKING** for any dashboard/alert filtering `sourcetype=macos:perf:snapshot` or
  relying on `MAC_PERF_SNAPSHOTS_DIR`.

### v0.3.1 (2026-07-08)

- **Pipeline file moved to Cribl's on-disk layout**: `default/pipelines/main.yml`
  (flat file in API-response shape) → `default/pipelines/main/conf.yml` (pipeline
  conf shape). The flat file was not loadable as pack pipeline config; routes
  referencing `main` would not have applied the pack's field stamping. No logic
  changes — functions are identical.
- **`default/pack.yml` corrected** to a pack manifest (was a stray `id: default`
  copied from route config).
- **CI fixed**: the reusable validate workflow reference now uses the renamed
  GitHub account owner (Actions `uses:` does not follow account-rename redirects;
  every run since the rename failed at workflow-file resolution). Renovate preset
  reference updated for the same reason.
- **Docs**: removed references to a private external repo from this public README;
  release commands in `CLAUDE.md` point at the current repo owner.
- **`.gitignore`**: locally-built `*.crbl` release artifacts are now ignored.

### v0.3.0 (2026-05-20)

- **Native 4.18 Sources adopted**:
  - `in_apple_unified_logs` (broad capture, `subsystem BEGINSWITH "com.apple"`)
  - `in_system_metrics` (host CPU/memory/disk/network + process)
- **Six exec inputs retired** in favor of the native Sources:
  - `macos-memory-pressure`, `macos-vm-stat`, `macos-disk-io`, `macos-process-top`,
    `macos-windowserver-health`, `macos-jetsam-events`
- **Architecture shift**: Edge now captures broadly; Stream-side pipelines handle
  per-use-case filtering. Anomaly detection for memory pressure / WindowServer pings /
  Jetsam events moves to Stream (out of scope for this pack release).
- **Three exec inputs retained**: `macos-thermal`, `macos-power-metrics`,
  `macos-power-battery` — no 4.18 native equivalent.
- **`minLogStreamVersion` bumped to `4.18.0`** (required for the new Source types).
- **BREAKING** for v0.2 dashboards filtering `sourcetype=macos:system:memory|windowserver|jetsam|process|diskio|vmstat`
  — those sourcetypes no longer exist on Edge. The equivalent data is now under
  `macos:unified_log` and `macos:system:metrics`; rebuild downstream dashboards accordingly.

### v0.2.0 (2026-04-29)

- **New File input** `mac-perf-snapshots` — tails NDJSON files emitted by an external
  snapshot collector. Reads `*.ndjson` from `$MAC_PERF_SNAPSHOTS_DIR`, parses each JSON line, sets `_time`
  from the event's `ts` field, routes to `index=mac_perf` with sourcetype
  `macos:perf:snapshot`.
- **Powermetrics retargeted** — the existing `macos-power-metrics` Exec input now writes
  to `index=mac_perf` with sourcetype `macos:perf:powermetrics`.
  **BREAKING** for any v0.1.0 dashboards or alerts that filtered
  `index=os sourcetype=macos:power:metrics`; update saved searches accordingly.
- **New `perf` streamtag** added to pack metadata.
- **Sample queries:**

  ```spl
  index=mac_perf sourcetype="macos:perf:snapshot" earliest=-1h | head 5
  index=mac_perf sourcetype="macos:perf:powermetrics" earliest=-1h | head 5
  ```

- **Prerequisites for v0.2.0:**
  - Splunk index `mac_perf` exists (provisioned by ansible-splunk).
  - Splunk add-on `VisiCore_TA_AI_Observability` includes `[macos:perf:snapshot]` and `[macos:perf:powermetrics]` props.
  - For the file input to have data: a snapshot collector must be installed on the Mac
    (e.g., as a per-user LaunchAgent).
  - Cribl Edge worker has `MAC_PERF_SNAPSHOTS_DIR` env var set.

### v0.1.0 (2026-04-18)

- **Initial release** of `cc-edge-the-mac-pack-io` — consolidates `cc-edge-macos-system`
  and `cc-edge-macos-power` (both predecessor repos archived).
- **Nine Exec inputs**:
  - System monitoring: WindowServer health (60s), memory pressure (60s),
    Jetsam events (5min), process stats (60s), disk I/O (60s), VM stats (60s),
    thermal status (60s)
  - Power monitoring: per-process energy metrics + thermal (300s),
    battery health (60s)
- **Anomaly detection** for: memory pressure, WindowServer timeouts, Jetsam events,
  battery health degradation, thermal pressure elevation.
- **Sourcetype namespaces**: `macos:system:*` and `macos:power:*`.

---

> Part of a [larger ecosystem of ~40 repos](https://docs.jacobpevans.com) — see how it all fits together.
