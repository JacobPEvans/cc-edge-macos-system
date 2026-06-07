# cc-edge-the-mac-pack-io

Cribl Edge Pack for comprehensive macOS system, power, and performance monitoring.

> Supersedes `cc-edge-macos-system` and `cc-edge-macos-power` — both repos are archived.

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
- **Producer**: Standalone Python collector (e.g., `nix-mac-performance/monitoring/collect-snapshot.py`)
  writing daily-rotated `<YYYY-MM-DD>.ndjson` files via a per-user LaunchAgent on a
  5-minute cadence
- **Time extraction**: `_time` is set from the `ts` field in each event (ISO 8601 UTC),
  not Cribl ingestion time
- **Requires**: Read access to the snapshot directory; no special privileges

## Network Health

Three sources together answer "is my internet actually good, and when it isn't, is the
problem my local network or the connection beyond it?" Reachability (`wan`) decomposes
latency into a LAN leg vs an internet leg; quality (`quality`) measures responsiveness
under load (bufferbloat); Wi-Fi (`wifi`) supplies the link-quality signal that explains
a bad local hop.

### WAN/LAN Reachability (`macos-wan-health`) — Exec Source

- **Interval**: 60 seconds
- **Command**: Bash running `ping` to the default gateway and public resolvers
  (`1.1.1.1`, `8.8.8.8`) — full command in [`default/inputs.yml`](default/inputs.yml)
- **Sourcetype**: `macos:network:wan`
- **Index**: `${INDEX}` (default `os`)
- **Captures**: Per-target packet loss % and round-trip latency (min/avg/max/stddev ms)
  for the gateway (LAN baseline) and each public resolver, as a `probes[]` array
- **Decomposition** (computed in pipeline from `probes[]`): the gateway probe is the
  local hop and the best external probe is the internet leg, yielding `lan_avg_ms`,
  `wan_avg_ms`, `internet_latency_ms` (= `wan_avg_ms − lan_avg_ms`),
  `lan_latency_share_pct`, reachability booleans, and a `network_health` verdict
  (`ok` / `lan_degraded` / `wan_degraded` / `internet_down` / `gateway_down`)
- **Thresholds** (env-tunable): `WAN_LAN_LAT_MS` (default `15`) flags an elevated local
  hop; `WAN_INET_LAT_MS` (default `50`) flags elevated internet latency
- **Why exec**: No native Source exposes end-to-end ping reachability (loss/latency)
  to external targets
- **Requires**: No special privileges

### Internet Quality (`macos-network-quality`) — Exec Source

- **Interval**: 1800 seconds (30 minutes)
- **Command**: `networkQuality -c` (Apple's built-in tool; `-c` emits JSON)
- **Sourcetype**: `macos:network:quality`
- **Index**: `${INDEX}` (default `os`)
- **Captures**: `responsiveness` (RPM — round-trips/min under load; higher is better,
  low = bufferbloat), `dl_throughput` / `ul_throughput` (bits/sec), `base_rtt` (ms),
  and `dl_bytes_transferred` / `ul_bytes_transferred` (bandwidth the test itself used)
- **WARNING — bandwidth**: each run is a *saturating* speed test that can move
  significant data (potentially GBs on a fast link). The 30-min interval bounds the
  cost; raise `interval` or set `disabled: true` in a local override to dial it back
- **Why exec**: No native Source runs an active responsiveness/throughput test
- **Requires**: No special privileges

### Wi-Fi Link Quality (`macos-network-wifi`) — Exec Source

- **Interval**: 60 seconds
- **Command**: `system_profiler -json SPAirPortDataType`
- **Sourcetype**: `macos:network:wifi`
- **Index**: `${INDEX}` (default `os`)
- **Captures**: Raw nested JSON for the active Wi-Fi interface. Per the
  Edge-captures / Stream-filters rule the structure ships as-is; the attribution
  fields live under
  `SPAirPortDataType[].spairport_airport_interfaces[].spairport_current_network_information`:
  `spairport_signal_noise` (`"-64 dBm / -93 dBm"` = RSSI / noise),
  `spairport_network_rate` (tx Mbps), `spairport_network_channel`,
  `spairport_network_phymode`
- **Why exec**: No native Source exposes Wi-Fi RSSI / noise / PHY rate
- **Requires**: No special privileges (no `sudo`); emits empty interface data on
  wired-only hosts

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
                                    | macos:power:battery | macos:network:wan
                                    | macos:network:quality | macos:network:wifi
                   index=mac_perf   sourcetype=macos:system:metrics | macos:perf:powermetrics
                                    | macos:perf:snapshot
```

## Anomaly Detection

Edge keeps anomaly detection ONLY for events that come from the exec sources
(powermetrics, battery, WAN/LAN reachability). Anomaly detection for memory pressure,
WindowServer ping timeouts, and Jetsam events moves to Stream — the broad Apple Unified
Logs / System Metrics streams already carry the underlying data; Stream pipelines
extract the signal.

| Condition | anomaly_reason | Detected at |
|-----------|----------------|-------------|
| Battery health < 80% | `battery_health_degraded` | Edge (this pack) |
| Thermal pressure not Nominal | `thermal_pressure_elevated` | Edge (this pack) |
| Local hop degraded (gateway loss or latency > `WAN_LAN_LAT_MS`) | `lan_degraded` | Edge (this pack) |
| Internet leg degraded (external loss or latency > `WAN_INET_LAT_MS`) | `wan_degraded` | Edge (this pack) |
| No external reachability | `internet_down` | Edge (this pack) |
| Gateway unreachable | `gateway_down` | Edge (this pack) |
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
| `macos:network:wan` | `${INDEX}` (default `os`) | Exec |
| `macos:network:quality` | `${INDEX}` (default `os`) | Exec |
| `macos:network:wifi` | `${INDEX}` (default `os`) | Exec |

The file-based snapshot input requires the `MAC_PERF_SNAPSHOTS_DIR` environment variable
to be set on the Cribl Edge worker, pointing at the directory where the snapshot
collector writes its NDJSON files
(e.g., `/Users/<you>/git/nix-mac-performance/main/monitoring/snapshots`).

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
  macos-network-quality:
    interval: 3600               # halve the speed-test bandwidth (or disabled: true)
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

### WAN/LAN Reachability Events (`macos:network:wan`)

| Field | Type | Description |
|-------|------|-------------|
| probes | array | Per-target results `{name, target, loss_pct, min_ms, avg_ms, max_ms, stddev_ms}` |
| probes[].name | string | `gateway`, `cloudflare`, or `google` |
| probes[].target | string | Probed IP (gateway resolved from the host default route) |
| probes[].loss_pct | number | Packet loss percentage for the probe |
| probes[].avg_ms | number | Mean round-trip latency (ms); `null` on 100% loss |
| lan_avg_ms | number | Gateway (local hop) mean latency — derived |
| lan_loss_pct | number | Gateway packet loss % — derived |
| wan_avg_ms | number | Best external (internet leg) mean latency — derived |
| wan_loss_pct | number | Best external packet loss % — derived |
| internet_latency_ms | number | `wan_avg_ms − lan_avg_ms` (≥ 0) — latency beyond the local hop |
| lan_latency_share_pct | number | Local hop's share of total latency (`lan/wan × 100`) |
| gateway_reachable | boolean | Gateway loss < 100% |
| internet_reachable | boolean | At least one external target loss < 100% |
| network_health | string | `ok` / `lan_degraded` / `wan_degraded` / `internet_down` / `gateway_down` |

### Internet Quality Events (`macos:network:quality`)

| Field | Type | Description |
|-------|------|-------------|
| responsiveness | number | Round-trips per minute under load (higher is better; low = bufferbloat) |
| dl_throughput | number | Download throughput (bits/sec) |
| ul_throughput | number | Upload throughput (bits/sec) |
| base_rtt | number | Idle base round-trip time (ms) |
| dl_bytes_transferred | number | Bytes the test downloaded (its own bandwidth cost) |
| ul_bytes_transferred | number | Bytes the test uploaded |
| interface_name | string | Interface the test ran over (e.g. `en0`) |

### Wi-Fi Link Quality Events (`macos:network:wifi`)

Raw `system_profiler` JSON; the attribution fields are nested under
`SPAirPortDataType[].spairport_airport_interfaces[].spairport_current_network_information`.

| Field (nested path leaf) | Type | Description |
|-------|------|-------------|
| spairport_signal_noise | string | `"<RSSI> dBm / <noise> dBm"` (e.g. `-64 dBm / -93 dBm`) |
| spairport_network_rate | number | Negotiated tx rate (Mbps) |
| spairport_network_channel | string | Channel, band, and width (e.g. `133 (6GHz, 160MHz)`) |
| spairport_network_phymode | string | Wi-Fi PHY mode (e.g. `802.11ax`) |

### Native Source Events

The native `apple_unified_logs` and `system_metrics` Sources emit Cribl-defined event
shapes — see Cribl's [macOS System Metrics Details](https://docs.cribl.io/edge/system-metrics-macos-output)
and [Apple Unified Logs Source](https://docs.cribl.io/edge/sources-apple-unified-logs)
reference docs for the full field listings. Edge passes them through unchanged; Stream
pipelines do downstream extraction.

### Common Fields (all sourcetypes)

| Field | Type | Description |
|-------|------|-------------|
| index | string | `os` for unified-log + thermal + battery + network; `mac_perf` for system metrics + powermetrics + snapshots |
| sourcetype | string | See namespace table above |
| host | string | Hostname of the Edge node |
| anomaly | boolean | `true` when anomaly detected (exec sources only — see Anomaly Detection) |
| anomaly_reason | string | Anomaly type identifier |

## Release Notes

### v0.4.0 (2026-06-06)

Network health — measure true internet quality and separate the local network's
contribution from the connection beyond it. Three script-free Exec inputs:

- **`macos-wan-health`** — WAN/LAN reachability: `ping` packet loss and round-trip
  latency (min/avg/max/stddev) to the default gateway and public resolvers
  (`1.1.1.1`, `8.8.8.8`), emitted as a `probes[]` array. 60s interval, sourcetype
  `macos:network:wan`, index `${INDEX}` (default `os`).
- **Pipeline decomposition** of `probes[]` into LAN vs internet: `lan_avg_ms`,
  `wan_avg_ms`, `internet_latency_ms`, `lan_latency_share_pct`, reachability booleans,
  and a `network_health` verdict (`ok` / `lan_degraded` / `wan_degraded` /
  `internet_down` / `gateway_down`). A non-`ok` verdict sets `anomaly=true` with
  `anomaly_reason` = the verdict. Thresholds env-tunable: `WAN_LAN_LAT_MS` (15),
  `WAN_INET_LAT_MS` (50).
- **`macos-network-quality`** — Apple `networkQuality -c`: responsiveness under load
  (RPM / bufferbloat) + down/up throughput. 1800s interval, sourcetype
  `macos:network:quality`. **Each run is a saturating speed test** — raise `interval`
  or disable via a local override to bound bandwidth.
- **`macos-network-wifi`** — `system_profiler -json SPAirPortDataType`: Wi-Fi
  RSSI/noise, tx rate, channel, PHY mode (no `sudo`). 60s interval, sourcetype
  `macos:network:wifi`. Raw JSON ships as-is; extract downstream per the
  Edge-captures / Stream-filters rule.
- **New `network` streamtag** added to pack metadata.

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
  snapshot collector (such as `nix-mac-performance/monitoring/collect-snapshot.py`).
  Reads `*.ndjson` from `$MAC_PERF_SNAPSHOTS_DIR`, parses each JSON line, sets `_time`
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
    (e.g., `nix-mac-performance` PR #2 LaunchAgent).
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
