# cc-edge-the-mac-pack-io

Cribl Edge Pack for comprehensive macOS system and power monitoring.

> Supersedes `cc-edge-macos-system` and `cc-edge-macos-power` — both repos are archived.

## Overview

Collects macOS system health and power data via native system tools using Cribl Edge Exec sources. Covers CPU, memory, disk, virtual memory, process stats, thermal state, WindowServer health, Jetsam events, per-process energy impact, and battery lifecycle. All events flow through Cribl Stream into Splunk (or any destination).

This pack is the macOS equivalent of Splunk's Splunk_TA_nix and Splunk_TA_windows — delivering comprehensive operating system telemetry via Cribl Edge instead of scripted inputs on the Splunk forwarder.

## Origin Story

This pack was born from a real incident: a 60-second MacBook UI freeze went undetected because WindowServer ping timeout events required manual log forensics to discover. The `macos-windowserver-health` source is the highest-priority data source in this pack, turning what was previously invisible into a first-class observable signal.

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
- **Command**: `powermetrics --samplers tasks,battery,cpu_power,gpu_power,ane_power,thermal --show-process-energy -f plist -n 1 -i 5000 | plutil -convert json -o - -`
- **Sourcetype**: `macos:power:metrics`
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

## Data Flow

```
memory_pressure / log / top / iostat / vm_stat / pmset / powermetrics / ioreg
    |
Cribl Edge (native macOS, /opt/cribl/)
    |  HEC
Cribl Stream
    |  HEC
Splunk -> index: os, sourcetype: macos:system:{type} or macos:power:{type}
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
index=os sourcetype=macos:system:* OR sourcetype=macos:power:* anomaly=true
| stats count by host, anomaly_reason, _time
```

## Usage

Once installed, the pack automatically collects data on the configured intervals. All system sources use `macos:system:*` sourcetypes; power/battery sources use `macos:power:*` sourcetypes. All events go to the `os` index.

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
| index | string | Always `os` |
| sourcetype | string | `macos:system:{type}` or `macos:power:{type}` |
| host | string | Hostname of the Edge node |
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

### v1.1.0 (2026-04-18)

- Merged `cc-edge-macos-power` into this pack (both predecessor repos archived)
- Renamed from `cc-edge-macos-system` to `cc-edge-the-mac-pack-io`
- Added `macos-power-metrics` input: per-process energy impact, CPU/GPU/ANE power, thermal pressure (300s, root required)
- Added `macos-power-battery` input: charge %, power source, cycle count, capacity, health % (60s)
- Extended pipeline: battery health calculation, top-10 energy processes, thermal pressure anomaly, battery health anomaly
- Sourcetype namespaces: `macos:system:*` (unchanged) and `macos:power:*` (new)

### v1.0.0 (2026-03-27)

- Initial release as `cc-edge-macos-system`
- Seven Exec sources: WindowServer health (60s), memory pressure (60s), Jetsam events (5min), process stats (60s), disk I/O (60s), VM stats (60s), thermal status (60s)
- Anomaly detection for memory pressure, WindowServer ping timeouts, and Jetsam events
