# Changelog

Release history for this pack. Newest first.

## v0.4.0 (2026-08-23)

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
- **New `macos-wired-memory` Exec input** — wired memory against the `iogpu.wired_limit_mb`
  ceiling is now reported; neither value was tracked before.
- **Sourcetype routing fixed**: the pipeline's sourcetype eval was a fallback chain that
  silently mislabeled any unmatched `datatype`; replaced with an explicit per-datatype
  map with no fallback.
- **Removed** the orphaned `mac-perf-snapshots` File input and its `macos:perf:snapshot`
  sourcetype — the external NDJSON collector that fed it was deliberately deleted.
  **BREAKING** for any dashboard/alert filtering `sourcetype=macos:perf:snapshot` or
  relying on `MAC_PERF_SNAPSHOTS_DIR`.

## v0.3.1 (2026-07-08)

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

## v0.3.0 (2026-05-20)

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

## v0.2.0 (2026-04-29)

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

## v0.1.0 (2026-04-18)

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
