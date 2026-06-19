# cc-edge-the-mac-pack-io

Cribl Edge Pack for comprehensive macOS system, power, and performance monitoring.

## Overview

Collects macOS telemetry on a Cribl Edge Node running natively on macOS. As of v0.3.0
the pack uses Cribl 4.18's native macOS Sources for broad host-metrics and Unified-Log
capture, falling back to Exec sources only for data the native Sources don't yet expose
(per-process energy via `powermetrics`, battery cycle/health via `ioreg`, and macOS
thermal pressure via `pmset`).

Design rule: **Edge captures broadly, Stream filters.** Source-level predicates that
narrow to specific use cases (WindowServer ping detection, Jetsam triage, security event
extraction, etc.) belong in Stream-side pipelines downstream of Edge — not in this pack.

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
- **Predicate**: `subsystem BEGINSWITH "com.apple"` — broad capture of every Apple
  subsystem (kernel, WindowServer, security, network, power, Spotlight, Time Machine,
  TCC, sandbox, launchservices, xpc, etc.). Override to `TRUEPREDICATE` for 3rd-party
  app logs as well.
- **Read mode**: `lastEntry` (new entries only on (re)start). Switch to `allEvents` for
  historical backfill.
- **Sourcetype** (assigned in pipeline): `macos:unified_log`
- **Index**: `os`
- **Replaces**: v0.2 exec sources `macos-windowserver-health` and `macos-jetsam-events`.
  Filtering for those specific signals now lives in Stream pipelines.

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
  piped through `plutil -convert json` — full command in [`default/inputs.yml`](default/inputs.yml)
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

### Perf Snapshots (`mac-perf-snapshots`) — File Source

- **Interval**: 30 seconds (file poll)
- **Source**: NDJSON files at `$MAC_PERF_SNAPSHOTS_DIR` (one event per line)
- **Sourcetype**: `macos:perf:snapshot`
- **Index**: `mac_perf`
- **Captures**: Aggregate host perf snapshot — load averages, memory totals, swap I/O,
  top CPU/RSS processes, zombie process tree, sleep assertions, 24h crash counts,
  listening ports, logged-in users, kernel_task CPU%
- **Producer contract**: Any external collector that writes daily-rotated
  `<YYYY-MM-DD>.ndjson` files (one event per line, each carrying an ISO 8601 UTC `ts`
  field) into `$MAC_PERF_SNAPSHOTS_DIR`. Typically driven by a per-user LaunchAgent on a
  5-minute cadence. The pack does not ship the collector — it consumes whatever NDJSON
  appears in that directory.
- **Time extraction**: `_time` is set from the `ts` field in each event (ISO 8601 UTC),
  not Cribl ingestion time
- **Requires**: Read access to the snapshot directory; no special privileges

## Data Flow

```text
Apple Unified Logging                                    Cribl native
   subsystem BEGINSWITH "com.apple"        ┐              4.18 Source
macOS kernel/system metrics                ┤
   CPU, memory, disk, network, processes   ┘
pmset / powermetrics / ioreg               ┐              Cribl Exec
                                           ┤              + File
collect-snapshot.py NDJSON files           ┘              Sources
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
                                    | macos:power:battery
                   index=mac_perf   sourcetype=macos:system:metrics | macos:perf:powermetrics
                                    | macos:perf:snapshot
```

## Anomaly Detection

Edge keeps anomaly detection ONLY for events that come from the exec sources
(powermetrics, battery). Anomaly detection for memory pressure, WindowServer ping
timeouts, and Jetsam events moves to Stream — the broad Apple Unified Logs / System
Metrics streams already carry the underlying data; Stream pipelines extract the signal.

| Condition | anomaly_reason | Detected at |
| --- | --- | --- |
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
| `macos:perf:snapshot` | `mac_perf` | File (NDJSON) |

The file-based snapshot input requires the `MAC_PERF_SNAPSHOTS_DIR` environment variable
to be set on the Cribl Edge worker, pointing at the directory where an external snapshot
collector writes its NDJSON files (e.g., `/Users/<you>/snapshots`).

To customize, create local overrides in `/opt/cribl/local/cc-edge-the-mac-pack-io/`:

```yaml
# /opt/cribl/local/cc-edge-the-mac-pack-io/inputs.yml
inputs:
  in_apple_unified_logs:
    predicate: 'TRUEPREDICATE'   # capture 3rd-party app logs too
  in_system_metrics:
    pollingInterval: 30          # increase metric frequency
  macos-power-metrics:
    disabled: true               # disable if root unavailable
  mac-perf-snapshots:
    disabled: true               # disable if no external collector is running
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
| --- | --- | --- |
| top_processes | array | Top 10 processes by energy impact `{name, pid, energy_impact}` |
| thermal_pressure | string | Thermal pressure state (`Nominal`, `Moderate`, `Heavy`, `Trapping`) |
| processor | object | CPU package power (mW), frequency per cluster |
| gpu | object | GPU power (mW) |
| ane_power | number | Apple Neural Engine power (mW) |

### Battery Events (`macos:power:battery`)

| Field | Type | Description |
| --- | --- | --- |
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
| --- | --- | --- |
| index | string | `os` for unified-log + thermal + battery; `mac_perf` for system metrics + powermetrics + snapshots |
| sourcetype | string | See namespace table above |
| host | string | Hostname of the Edge node |
| anomaly | boolean | `true` when anomaly detected (exec sources only — see Anomaly Detection) |
| anomaly_reason | string | Anomaly type identifier |

## Release Notes

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
  snapshot collector. Reads `*.ndjson` from `$MAC_PERF_SNAPSHOTS_DIR`, parses each JSON
  line, sets `_time` from the event's `ts` field, routes to `index=mac_perf` with
  sourcetype `macos:perf:snapshot`.
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
  - Splunk index `mac_perf` exists.
  - The receiving Splunk add-on includes `[macos:perf:snapshot]` and
    `[macos:perf:powermetrics]` props/sourcetypes.
  - For the file input to have data: an external snapshot collector must be installed on
    the Mac, writing NDJSON to `$MAC_PERF_SNAPSHOTS_DIR`.
  - Cribl Edge worker has `MAC_PERF_SNAPSHOTS_DIR` env var set.

### v0.1.0 (2026-04-18)

- **Initial release** of `cc-edge-the-mac-pack-io` — a single pack covering macOS system
  and power telemetry.
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
