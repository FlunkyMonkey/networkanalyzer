# Source-to-Signal Mapping

This document maps every known lab source to the signals it provides, the requirements it supports, and the telemetry plane it feeds.

## Source Inventory

### MikroTik CRS328

| Attribute | Value |
|---|---|
| **Signals** | Per-port traffic counters (bytes in/out, packets, errors, discards), port link status, PoE status, VLAN assignment |
| **Requirements supported** | Bandwidth utilization by switch interfaces |
| **Telemetry plane** | Infrastructure / Telemetry |
| **Collection method** | SNMP polling (IF-MIB, MIKROTIK-MIB). RouterOS supports SNMPv2c/v3. |
| **Entity relationships** | `switch` -> `switch_interface` -> `vlan`; switch_interface connects to `ap`, `proxmox_host`, `firewall`, or endpoint `mac`/`ip` |
| **Limitations** | No native flow export. No L7 visibility. SNMP polling interval limits granularity (typically 60s minimum). No per-client breakdown — only per-port. |
| **Phase 1 required** | Yes — primary source for switch interface utilization |

### UniFi Controller / APs / WLAN Clients

| Attribute | Value |
|---|---|
| **Signals** | Per-AP traffic and client counts, per-SSID metrics, per-WLAN-client bandwidth (rx/tx bytes, signal, connection time), AP channel utilization, client roaming events |
| **Requirements supported** | Bandwidth utilization by WLAN clients, AP interface metrics |
| **Telemetry plane** | Infrastructure / Telemetry |
| **Collection method** | UniFi Controller API (REST). Polling `/api/s/{site}/stat/sta` for client stats, `/api/s/{site}/stat/device` for AP stats. Alternatively, unifi-poller/Telegraf UniFi input plugin for Prometheus-format metrics. |
| **Entity relationships** | `ap` -> `ssid` -> client (`mac`, `ip`, `hostname`); AP connects to `switch_interface` |
| **Limitations** | API rate limits not well documented. Historical data in controller is limited (typically 24h–7d depending on stat type). Client `hostname` may be absent or generic. Signal strength is per-client but not per-flow. |
| **Phase 1 required** | Yes — primary source for WLAN client bandwidth |

### Firewalla Gold SE

| Attribute | Value |
|---|---|
| **Signals** | WAN throughput, per-device bandwidth, flow logs (src/dst IP, port, protocol, bytes), DNS query logs, IDS/IPS alerts, app/category classification (Firewalla-native), blocked connections, VPN tunnel stats |
| **Requirements supported** | Top talkers, destination/website analysis, app/category classification, security analytics, WAN utilization |
| **Telemetry plane** | Flow Analytics (flows, DNS), Infrastructure / Telemetry (WAN interface), Security (IDS alerts) |
| **Collection method** | Firewalla API (local REST, requires token). Flow/DNS logs may be exportable via syslog if configured. Firewalla MSP API for cloud-managed features. Exact export capabilities need in-cluster validation. |
| **Entity relationships** | `wan` -> `firewall`; firewall sees `ip` -> `dns_name`, `app`, `category`, `country`; connects to `router` downstream |
| **Limitations** | API is not publicly documented — capabilities need reverse-engineering or Firewalla community resources. Export granularity and rate limits unknown. Native app/category classification may not match platform taxonomy. May not support continuous streaming — likely poll-based. |
| **Phase 1 required** | No — but high value. Needed for top talkers and destination analysis (Phase 4). API feasibility should be validated in Phase 1. |

### Proxmox Cluster / Nodes / VMs

| Attribute | Value |
|---|---|
| **Signals** | Per-VM network bytes (in/out), per-node network interface counters, VM lifecycle events, node resource utilization |
| **Requirements supported** | Bandwidth utilization by Proxmox VMs, compute asset visibility |
| **Telemetry plane** | Infrastructure / Telemetry |
| **Collection method** | Proxmox API (`/api2/json/nodes/{node}/qemu/{vmid}/status/current` for VM net stats). Alternatively, SNMP on Proxmox hosts (IF-MIB for host interfaces), or Telegraf with Proxmox input plugin, or Prometheus proxmox-ve-exporter. |
| **Entity relationships** | `proxmox_host` -> `vm` -> `ip`/`mac`; proxmox_host connects to `switch_interface`; VM may host `k8s_node` |
| **Limitations** | Per-VM counters are aggregate (total bytes since boot) — need delta calculation. No per-flow visibility at VM level without additional tooling. Bridge interface counters may double-count if VMs share bridges. |
| **Phase 1 required** | Yes — required for Proxmox VM bandwidth utilization |

### Kubernetes Cluster

| Attribute | Value |
|---|---|
| **Signals** | Pod-to-pod traffic (bytes, packets, connections), service-to-service communication, DNS queries, L7 protocol metadata (HTTP paths, gRPC methods), network policy enforcement, namespace traffic aggregates |
| **Requirements supported** | Pod/workload/service visibility, path-style visualization, L7 drill-down |
| **Telemetry plane** | Kubernetes Network Visibility |
| **Collection method** | eBPF-based agent (Hubble/Cilium if Cilium CNI is in use, otherwise Pixie or Retina). If Cilium is the CNI, Hubble is native. Otherwise, a standalone eBPF agent is needed. |
| **Entity relationships** | `cluster` -> `namespace` -> `workload` -> `pod` -> `service`; pod `ip` correlates with flow analytics plane; `k8s_node` maps to `proxmox_host`/`vm` |
| **Limitations** | eBPF agent overhead on nodes. L7 parsing depends on protocol support in the chosen tool. Encrypted traffic (mTLS) may require integration with service mesh for decrypted metadata. Requires privileged DaemonSet. |
| **Phase 1 required** | No — Phase 5 |

### Softflowd Exporters (Proxmox Hosts)

| Attribute | Value |
|---|---|
| **Signals** | NetFlow v5/v9 or IPFIX records: src/dst IP, src/dst port, protocol, bytes, packets, timestamps, TCP flags |
| **Requirements supported** | Top talkers, destination analysis, port breakdown, flow analytics |
| **Telemetry plane** | Flow Analytics |
| **Collection method** | Already running on Proxmox hosts. Flows are exported to a collector endpoint. Need a flow collector (e.g., GoFlow2, pmacct, nfdump, or Elastiflow) deployed in-cluster to receive and store flows. |
| **Entity relationships** | Flow `src`/`dst` -> `ip` -> correlates with `vm`, `pod`, `mac`, `hostname` via enrichment; `port` -> `app` (via classification); `dst` -> `dns_name`, `country`, `asn` (via enrichment) |
| **Limitations** | Softflowd sees traffic at the Proxmox host bridge level — may not distinguish individual VM-to-VM traffic on the same host cleanly. No app/category classification natively — needs enrichment. No L7 metadata. Sampling rate and export interval affect granularity. |
| **Phase 1 required** | No — Phase 4. But these are already running, so collector deployment is the main gating item. |

### DNS Sources

| Attribute | Value |
|---|---|
| **Signals** | DNS query/response logs: queried name, response IP, client IP, query type, NXDOMAIN/SERVFAIL counts, latency |
| **Requirements supported** | Destination/website analysis (dns_name enrichment), DNS anomaly detection (security analytics), entity correlation (IP -> dns_name) |
| **Telemetry plane** | Flow Analytics (enrichment), Security (anomaly detection) |
| **Collection method** | Depends on DNS infrastructure: if using Pi-hole or AdGuard, their APIs/logs. If using CoreDNS in K8s, its query logging. Firewalla DNS logs (if exportable). Passive DNS capture via softflowd on port 53 flows. |
| **Entity relationships** | `ip` -> `dns_name`; `dns_name` -> `category`, `country` (via enrichment databases) |
| **Limitations** | DNS-over-HTTPS/TLS from clients bypasses local DNS — invisible unless intercepted at firewall. Passive DNS from flow data gives query/response IPs but not the queried name. Need active DNS logging, not just flow-level. |
| **Phase 1 required** | No — but DNS enrichment is critical for Phase 4 (destination analysis). DNS source availability should be inventoried in Phase 1. |

---

## Requirements-to-Signal Mapping

This section maps each approved requirement to the sources and signals that satisfy it.

### Top Talkers

| Attribute | Value |
|---|---|
| **Primary sources** | Softflowd exporters (flow bytes by src/dst), Firewalla (per-device bandwidth) |
| **Enrichment** | DNS (IP -> name), GeoIP (IP -> country/ASN) |
| **Plane** | Flow Analytics |
| **Phase** | 4 |

### What Websites / Destinations Server X Is Connecting To

| Attribute | Value |
|---|---|
| **Primary sources** | Softflowd (dst IPs per src), Firewalla (dst domains per device), DNS logs (query -> response mapping) |
| **Enrichment** | DNS reverse/passive (IP -> domain), GeoIP, app/category classification |
| **Plane** | Flow Analytics |
| **Phase** | 4 |

### Bandwidth Utilization by Switch Interfaces

| Attribute | Value |
|---|---|
| **Primary source** | MikroTik CRS328 via SNMP (IF-MIB per-port counters) |
| **Enrichment** | Interface-to-device mapping (which device is on which port) |
| **Plane** | Infrastructure / Telemetry |
| **Phase** | 3 |

### Bandwidth Utilization by WLAN Clients

| Attribute | Value |
|---|---|
| **Primary source** | UniFi Controller API (per-client rx/tx bytes) |
| **Enrichment** | Client hostname, AP and SSID context |
| **Plane** | Infrastructure / Telemetry |
| **Phase** | 3 |

### Bandwidth Utilization by Proxmox VMs

| Attribute | Value |
|---|---|
| **Primary source** | Proxmox API (per-VM netin/netout counters) |
| **Enrichment** | VM name -> hostname, VM -> K8s node mapping |
| **Plane** | Infrastructure / Telemetry |
| **Phase** | 3 |

### Categories / Apps / Countries / Ports

| Attribute | Value |
|---|---|
| **Primary sources** | Softflowd (port, protocol), Firewalla (native app/category) |
| **Enrichment** | GeoIP (country, ASN), DNS (domain -> category via classification DB), port -> app mapping |
| **Plane** | Flow Analytics |
| **Phase** | 4 |

### Pod / Workload / Service Visibility

| Attribute | Value |
|---|---|
| **Primary source** | Kubernetes eBPF agent (Hubble/Cilium, Pixie, or Retina) |
| **Enrichment** | K8s metadata (namespace, labels, service selectors) |
| **Plane** | Kubernetes Network Visibility |
| **Phase** | 5 |

### Path-Style Visualization

| Attribute | Value |
|---|---|
| **Sources** | All planes — requires entity correlation across infra, flow, and K8s |
| **Key links** | switch_interface -> IP -> flow -> dns_name; IP -> VM -> K8s node -> pod |
| **Plane** | Correlation / UX |
| **Phase** | 6 |

### Degraded-State Banners

| Attribute | Value |
|---|---|
| **Sources** | Health/readiness endpoints from all collectors, freshness metadata per pipeline |
| **Implementation** | Each collector reports last-successful-poll timestamp; UX plane compares against freshness threshold |
| **Plane** | Correlation / UX (display), all planes (data) |
| **Phase** | 3 (backend), 6 (UI) |

### Security Analytics

| Attribute | Value |
|---|---|
| **Primary sources** | Firewalla IDS/IPS alerts, flow data (anomaly detection), DNS logs (tunneling/DGA detection) |
| **Enrichment** | Threat intelligence feeds, first-seen tracking |
| **Plane** | Flow Analytics + Security (cross-cutting) |
| **Phase** | 7 |

### Incident Preservation

| Attribute | Value |
|---|---|
| **Mechanism** | Alert triggers a preservation window; full-granularity data in the time range is marked exempt from retention aging |
| **Sources** | All planes — any data within the preservation window is retained |
| **Plane** | Cross-cutting (retention layer) |
| **Phase** | 8 |

---

## Open Implementation Decisions

These decisions need in-cluster validation before implementation. They do not reopen approved architecture or product choices.

### 1. Firewalla API Feasibility

**Question:** What data can be reliably exported from the Firewalla Gold SE via its local API?

**Why it matters:** Firewalla is the richest potential source for top talkers, destination analysis, and app/category classification. If its API is limited, flow analytics will depend more heavily on softflowd + enrichment.

**Validation needed:** Enumerate available API endpoints, confirm flow/DNS log export, measure rate limits, assess data granularity.

### 2. DNS Logging Source

**Question:** Which DNS resolver(s) are in use, and can they export query logs?

**Why it matters:** IP-to-domain mapping is critical for the "what destinations is X connecting to" requirement. Without DNS logs, enrichment falls back to reverse DNS and passive inference.

**Validation needed:** Identify DNS infrastructure (Pi-hole, AdGuard, CoreDNS, Firewalla DNS, router DNS). Confirm log export capabilities.

### 3. Softflowd Export Configuration

**Question:** What is the current softflowd configuration (export format, sampling rate, active/inactive timeouts)?

**Why it matters:** Determines flow collector requirements and affects data granularity for top talker analysis.

**Validation needed:** Check softflowd config on each Proxmox host. Confirm export format (NetFlow v5/v9 or IPFIX) and target endpoint.

### 4. Kubernetes CNI and eBPF Availability

**Question:** Is Cilium the CNI, or is a different CNI in use? Is eBPF available on the kernel version?

**Why it matters:** Determines whether Hubble (Cilium-native) can be used or whether a standalone eBPF agent is needed.

**Validation needed:** Check CNI plugin, kernel version (`uname -r`), and eBPF support.

### 5. Time-Series Storage Selection

**Question:** Which time-series database should be used for metrics and flow storage?

**Why it matters:** Affects retention, rollup, and query capabilities across all planes. Candidates include Prometheus + Thanos/Mimir (metrics), ClickHouse (flows), VictoriaMetrics, or InfluxDB.

**Validation needed:** Evaluate storage footprint for expected data volume, rollup support, query performance, and Ceph PVC compatibility. This is an implementation decision, not an architecture decision — the approved retention model (30d granular, daily rollups, multi-year) constrains the choice.
