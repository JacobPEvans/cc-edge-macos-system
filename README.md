# cc-edge-macos-system

Cribl Edge Pack for macOS system health monitoring and anomaly detection.

## Overview

Collects macOS system health data via native system tools (`memory_pressure`, `log`, `top`, `iostat`, `vm_stat`, `pmset`) using Cribl Edge Exec sources. Designed to detect UI freezes (WindowServer ping timeouts), memory pressure events, Jetsam kills, and other system anomalies. Flows through Cribl Stream into Splunk (or any destination) for analysis and alerting.

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
- **Requires**: Root privileges (top requires elevated permissions for process enumeration)

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

## Data Flow

```
memory_pressure / log / top / iostat / vm_stat / pmset  (Exec Sources)
    |
Cribl Edge (native macOS, /opt/cribl/)
    |  HEC
Cribl Stream
    |  HEC
Splunk -> index: os, sourcetype: macos:system:{type}
```

## Anomaly Detection

The pipeline automatically flags anomalous events with `anomaly=true` and `anomaly_reason`:

| Condition | anomaly_reason |
|-----------|----------------|
| Memory free < 10% | `memory_pressure_critical` |
| Any WindowServer ping timeout | `windowserver_ping_timeout` |
| Recent Jetsam event detected | `jetsam_event_detected` |

Use these fields in Splunk (or any destination) to build alerts:

```spl
index=os sourcetype=macos:system:* anomaly=true
| stats count by host, anomaly_reason, _time
```

## Usage

Once installed, the pack automatically collects data on the configured intervals. All sources route events to the `os` index with `macos:system:*` sourcetypes.

To customize, create local overrides in `/opt/cribl/local/cc-edge-macos-system/`:

```yaml
# /opt/cribl/local/cc-edge-macos-system/inputs.yml
inputs:
  macos-windowserver-health:
    interval: 30  # Check more frequently for UI freezes
  macos-process-top:
    disabled: true  # Disable if top privileges unavailable
```

## Installation

1. Copy pack to Cribl Edge or install via API:

   ```bash
   # Copy .crbl file to Edge
   cp cc-edge-macos-system.crbl /opt/cribl/state/packs/

   # Install via API
   curl -X POST http://localhost:9000/api/v1/packs \
     -H "Content-Type: application/json" \
     -d '{"source":"cc-edge-macos-system.crbl"}'
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

### Common Fields (all sourcetypes)

| Field | Type | Description |
|-------|------|-------------|
| index | string | Always `os` |
| sourcetype | string | `macos:system:{type}` |
| host | string | Hostname of the Edge node |
| anomaly | boolean | `true` when anomaly detected (absent otherwise) |
| anomaly_reason | string | Anomaly type identifier (absent when no anomaly) |

## Future Roadmap

This pack aims to become the definitive macOS system monitoring pack for Cribl Edge. Planned future sources include:

- **Network statistics** (`netstat`, `nettop`, interface metrics)
- **Firewall status** (`socketfilterfw --getglobalstate`)
- **Login/auth events** (`log show` with auth predicates)
- **Software update status** (`softwareupdate --list`)
- **FileVault encryption status** (`fdesetup status`)
- **SIP status** (`csrutil status`)
- **Spotlight indexing** (`mdutil -s /`)
- **Time Machine status** (`tmutil status`)
- **Launch daemons/agents** inventory
- **Open file descriptors** (`lsof` summary)
- **CPU microarchitecture counters** (`powermetrics --samplers cpu_power`)
- **Disk space** (`df -h`)
- **System uptime and load** (`uptime`, `sysctl`)
- **Kernel parameters** (`sysctl -a`)
- **Installed software** (`system_profiler SPApplicationsDataType`)

## Release Notes

### v1.0.0 (2026-03-27)

- Initial release
- Seven Exec sources: WindowServer health (60s), memory pressure (60s), Jetsam events (5min), process stats (60s), disk I/O (60s), VM stats (60s), thermal status (60s)
- Pipeline with field extraction for memory pressure and WindowServer events
- Anomaly detection for memory pressure, WindowServer ping timeouts, and Jetsam events
- All events indexed to `os` with `macos:system:*` sourcetypes
