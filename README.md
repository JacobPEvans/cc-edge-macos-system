# cc-edge-the-mac-pack-io

Cribl Edge Pack for comprehensive macOS system, power, and performance monitoring.

> Supersedes `cc-edge-macos-system` and `cc-edge-macos-power` — both repos are archived.

## Overview

Collects macOS system health, power, and performance data via native system tools using Cribl Edge
Exec sources, plus full host snapshots tailed from NDJSON files via Cribl File sources.

Covers CPU, memory, disk, virtual memory, process stats, thermal state, WindowServer health,
Jetsam events, per-process energy impact, ANE/GPU/CPU power, battery lifecycle, and aggregate host
perf snapshots (load, RSS top, zombies, sleep assertions, crashes, listening ports).

Events flow through Cribl Stream into Splunk (or any destination).

This pack is the macOS equivalent of Splunk's Splunk_TA_nix and Splunk_TA_windows — delivering
comprehensive operating system telemetry via Cribl Edge instead of scripted inputs on the Splunk
forwarder.

## Origin Story

This pack was born from a real incident: a 60-second MacBook UI freeze went undetected because
WindowServer ping timeout events required manual log forensics to discover.

The `macos-windowserver-health` source is the highest-priority data source in this pack, turning
what was previously invisible into a first-class observable signal.

## Data Sources

### WindowServer Health (`macos-windowserver-health`) — PRIMARY

- **Interval**: 60 seconds
- **Command**: `/usr/bin/log show --last 65s` filtering for WindowServer ping events
- **Sourcetype**: `macos:system:windowserver`
- **Captures**: WindowServer ping timeout events indicating UI freezes, affected PIDs
- **Anomaly**: Any ping timeout event sets `anomaly=true`
- **Requires**: No special privileges

### Memory Pressure (`macos-memory-pressure`)

- **Interval**: 60 seconds
- **Command**: `memory_pressure`
- **Sourcetype**: `macos:system:memory`
- **Captures**: Pages free/active/inactive/wired/compressed, swap I/O, system-wide free percentage
- **Anomaly**: `memory_free_pct < 10` sets `anomaly=true`
- **Requires**: No special privileges

### Jetsam Events (`macos-jetsam-events`)

- **Interval**: 300 seconds (5 minutes)
- **Command**: Bash script to find and parse latest JetsamEvent `.ips` file
- **Sourcetype**: `macos:system:jetsam`
- **Captures**: Jetsam kill events including largest process, memory page counts, killed process list
- **Anomaly**: Presence of a recent Jetsam event sets `anomaly=true`
- **Requires**: Read access to `/Library/Logs/DiagnosticReports/`

### Process Stats (`macos-process-top`)

- **Interval**: 60 seconds
- **Command**: `top -l 1 -n 20 -stats pid,command,cpu,mem,rprvt,purg,vsize,threads`
- **Sourcetype**: `macos:system:process`
- **Captures**: Top 20 processes by CPU with memory, resident private, purgeable, virtual size, and thread counts
- **Requires**: Root privileges

### Disk I/O (`macos-disk-io`)

- **Interval**: 60 seconds
- **Command**: `iostat -d -c 2 -w 1`
- **Sourcetype**: `macos:system:diskio`
- **Captures**: Disk throughput (KB/t), transactions per second (tps), bandwidth (MB/s)
- **Requires**: No special privileges

### VM Statistics (`macos-vm-stat`)

- **Interval**: 60 seconds
- **Command**: `vm_stat`
- **Sourcetype**: `macos:system:vmstat`
- **Captures**: Detailed virtual memory statistics including page faults, compressor activity, purgeable pages, copy-on-write, swap I/O
- **Requires**: No special privileges

### Thermal Status (`macos-thermal`)

- **Interval**: 60 seconds
- **Command**: `pmset -g therm`
- **Sourcetype**: `macos:system:thermal`
- **Captures**: Thermal warning level, performance warning level, CPU power status
- **Requires**: No special privileges

### Power Metrics (`macos-power-metrics`)

- **Interval**: 300 seconds (5 minutes)
- **Command**: `powermetrics --samplers tasks,battery,cpu_power,gpu_power,ane_power,thermal ...`
  piped through `plutil -convert json` — full command in [`default/inputs.yml`](default/inputs.yml)
- **Sourcetype**: `macos:perf:powermetrics` (changed in v0.2.0; was `macos:power:metrics` in v0.1.0)
- **Index**: `mac_perf` (changed in v0.2.0; was `os` in v0.1.0)
- **Captures**: Per-process energy impact (top 10), CPU package power (mW), GPU power, ANE power, thermal pressure state, processor frequency
- **Anomaly**: `thermal_pressure !== 'Nominal'` sets `anomaly=true`
- **Requires**: Root privileges

### Battery Health (`macos-power-battery`)

- **Interval**: 60 seconds
- **Command**: Bash combining `pmset -g batt` + `ioreg -r -c AppleSmartBattery`
- **Sourcetype**: `macos:power:battery`
- **Captures**: Charge percentage, power source (AC/Battery), charging state, cycle count, max/design/current capacity, temperature, voltage
- **Derived**: `battery_health_percent = max_capacity / design_capacity × 100`
- **Anomaly**: `battery_health_percent < 80` sets `anomaly=true`
- **Requires**: No special privileges

### Perf Snapshots (`mac-perf-snapshots`) — File Source

- **Interval**: 30 seconds (file poll)
- **Source**: NDJSON files at `$MAC_PERF_SNAPSHOTS_DIR` (one event per line)
- **Sourcetype**: `macos:perf:snapshot`
- **Index**: `mac_perf`
- **Captures**: Aggregate host perf snapshot — load averages, memory totals, swap I/O,
  top CPU/RSS processes, zombie process tree, sleep assertions, 24h crash counts,
  listening ports, logged-in users, kernel_task CPU%
- **Producer**: Standalone Python collector (e.g., `nix-mac-performance/monitoring/collect-snapshot.py`)
  writing daily-rotated `<YYYY-MM-DD>.ndjson` files via a per-user LaunchAgent on a 5-minute cadence
- **Time extraction**: `_time` is set from the `ts` field in each event (ISO 8601 UTC), not Cribl ingestion time
- **Requires**: Read access to the snapshot directory; no special privileges

## Data Flow

```text
memory_pressure / log / top / iostat / vm_stat / pmset / powermetrics / ioreg
    |  (Cribl Exec sources)
    +----------------------------+
collect-snapshot.py NDJSON files  |
    |  (Cribl File source)       |
    +----------------------------+
                                  |
                  Cribl Edge (native macOS, /opt/cribl/)
                                  |  HEC
                            Cribl Stream
                                  |  HEC
                                  v
        Splunk:
          index=os         sourcetype=macos:system:* | macos:power:battery
          index=mac_perf   sourcetype=macos:perf:powermetrics | macos:perf:snapshot
```

## Anomaly Detection

The pipeline automatically flags anomalous events with `anomaly=true` and `anomaly_reason`:

| Condition | anomaly_reason |
|-----------|----------------|
| Memory free < 10% | `memory_pressure_critical` |
| Any WindowServer ping timeout | `windowserver_ping_timeout` |
| Recent Jetsam event detected | `jetsam_event_detected` |
| Battery health < 80% | `battery_health_degraded` |
| Thermal pressure not Nominal | `thermal_pressure_elevated` |

Use these fields in Splunk (or any destination) to build alerts:

```spl
(index=os OR index=mac_perf) (sourcetype=macos:system:* OR sourcetype=macos:power:* OR sourcetype=macos:perf:*) anomaly=true
| stats count by host, anomaly_reason, _time
```

## Usage

Once installed, the pack automatically collects data on the configured intervals.

| Sourcetype namespace | Index | Source type |
| --- | --- | --- |
| `macos:system:*` | `os` | Exec |
| `macos:power:battery` | `os` | Exec |
| `macos:perf:powermetrics` | `mac_perf` | Exec |
| `macos:perf:snapshot` | `mac_perf` | File (NDJSON) |

The file-based snapshot input requires the `MAC_PERF_SNAPSHOTS_DIR` environment variable to be
set on the Cribl Edge worker, pointing at the directory where the snapshot collector writes its
NDJSON files (e.g., `/Users/<you>/git/nix-mac-performance/main/monitoring/snapshots`).

To customize, create local overrides in `/opt/cribl/local/cc-edge-the-mac-pack-io/`:

```yaml
# /opt/cribl/local/cc-edge-the-mac-pack-io/inputs.yml
inputs:
  macos-windowserver-health:
    interval: 30  # Check more frequently for UI freezes
  macos-process-top:
    disabled: true  # Disable if root unavailable
  macos-power-metrics:
    disabled: true  # Disable if root unavailable
  mac-perf-snapshots:
    disabled: true  # Disable if no external collector is running
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

### Memory Events (`macos:system:memory`)

| Field | Type | Description |
|-------|------|-------------|
| pages_free | number | Free memory pages |
| pages_active | number | Active memory pages |
| pages_inactive | number | Inactive memory pages |
| pages_wired | number | Wired (non-pageable) memory pages |
| pages_compressed | number | Pages used by compressor |
| memory_free_pct | number | System-wide memory free percentage |

### WindowServer Events (`macos:system:windowserver`)

| Field | Type | Description |
|-------|------|-------------|
| ping_timeout_events | number | Count of ping timeout events in sample window |
| ping_timeout_pids | array | PIDs of processes that timed out on ping response |

### Jetsam Events (`macos:system:jetsam`)

| Field | Type | Description |
|-------|------|-------------|
| largestProcess | string | Process name consuming the most memory at time of Jetsam |
| memoryStatus | object | System memory status including page counts and compressor stats |
| processes | array | List of processes with memory details and states |

### Power Metrics Events (`macos:power:metrics`)

| Field | Type | Description |
|-------|------|-------------|
| top_processes | array | Top 10 processes by energy impact `{name, pid, energy_impact}` |
| thermal_pressure | string | Thermal pressure state (`Nominal`, `Moderate`, `Heavy`, `Trapping`) |
| processor | object | CPU package power (mW), frequency per cluster |
| gpu | object | GPU power (mW) |
| ane_power | number | Apple Neural Engine power (mW) |

### Battery Events (`macos:power:battery`)

| Field | Type | Description |
|-------|------|-------------|
| charge_percent | number | Current charge percentage |
| power_source | string | `AC` or `Battery` |
| charging_state | string | `charging`, `discharging`, `charged`, or `unknown` |
| cycle_count | number | Battery charge cycle count |
| max_capacity | number | Current maximum capacity (mAh) |
| design_capacity | number | Original design capacity (mAh) |
| battery_health_percent | number | Derived: `max_capacity / design_capacity × 100` |
| temperature | number | Battery temperature (hundredths of °C) |
| voltage | number | Battery voltage (mV) |

### Common Fields (all sourcetypes)

| Field | Type | Description |
|-------|------|-------------|
| index | string | `os` for system + battery sources; `mac_perf` for perf sources |
| sourcetype | string | `macos:system:{type}`, `macos:power:battery`, `macos:perf:powermetrics`, or `macos:perf:snapshot` |
| host | string | Hostname of the Edge node (or the snapshot event's `host` for file-source events) |
| anomaly | boolean | `true` when anomaly detected (absent otherwise) |
| anomaly_reason | string | Anomaly type identifier (absent when no anomaly) |

## Future Roadmap

- **Network statistics** (`netstat`, `nettop`, interface metrics)
- **Firewall status** (`socketfilterfw --getglobalstate`)
- **Login/auth events** (`log show` with auth predicates)
- **Software update status** (`softwareupdate --list`)
- **FileVault encryption status** (`fdesetup status`)
- **SIP status** (`csrutil status`)
- **Disk space** (`df -h`)
- **System uptime and load** (`uptime`, `sysctl`)
- **Open file descriptors** (`lsof` summary)

## Release Notes

### v0.2.0 (2026-04-29)

- **New File input** `mac-perf-snapshots` — tails NDJSON files emitted by an external snapshot
  collector (such as `nix-mac-performance/monitoring/collect-snapshot.py`).
  Reads `*.ndjson` from `$MAC_PERF_SNAPSHOTS_DIR`, parses each JSON line, sets `_time` from the
  event's `ts` field, routes to `index=mac_perf` with sourcetype `macos:perf:snapshot`.
- **Powermetrics retargeted** — the existing `macos-power-metrics` Exec input now writes to
  `index=mac_perf` with sourcetype `macos:perf:powermetrics`.
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
  - For the file input to have data: a snapshot collector must be installed on the Mac (e.g., `nix-mac-performance` PR #2 LaunchAgent).
  - Cribl Edge worker has `MAC_PERF_SNAPSHOTS_DIR` env var set.

### v0.1.0 (2026-04-18)

- **Initial release** of `cc-edge-the-mac-pack-io` — consolidates `cc-edge-macos-system` and `cc-edge-macos-power` (both predecessor repos archived)
- **Nine Exec inputs**:
  - System monitoring: WindowServer health (60s), memory pressure (60s), Jetsam events (5min),
    process stats (60s), disk I/O (60s), VM stats (60s), thermal status (60s)
  - Power monitoring: per-process energy metrics + thermal (300s), battery health (60s)
- **Anomaly detection** for: memory pressure, WindowServer timeouts, Jetsam events, battery health degradation, thermal pressure elevation
- **Sourcetype namespaces**: `macos:system:*` and `macos:power:*`
