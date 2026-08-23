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

## Data sources

Six inputs: Apple Unified Logs, system metrics, thermal status, power
metrics, battery health, wired memory, and crash reports. Per-source
detail — what each one collects, which binaries it calls, and the
sourcetype it lands under — is in [`docs/SOURCES.md`](docs/SOURCES.md).

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

## Release notes

See [`CHANGELOG.md`](CHANGELOG.md).
