# Infrastructure Telemetry Plane

## Component Stack

| Component | Version | Purpose | Deployed By |
|---|---|---|---|
| Grafana (standalone) | 10.5.15 | Visualization and dashboards | This repo |
| prometheus-snmp-exporter | 9.13.1 | SNMP polling proxy for MikroTik CRS328 | This repo |
| unpoller | 2.39.0 | UniFi controller/AP/client metrics exporter | This repo |
| prometheus-pve-exporter | 3.5.3 | Proxmox VE node and VM metrics exporter | This repo |
| Prometheus | (existing) | Scrape backend — 7d retention | Cluster infra (`monitoring` namespace) |
| Prometheus Operator | (existing) | ServiceMonitor CRD controller | Cluster infra (`monitoring` namespace) |

### Architecture (Wave 1 Pivot)

This repo does **not** deploy Prometheus or the Prometheus Operator. The cluster already has both in the `monitoring` namespace. This repo deploys:

- **Exporters** (SNMP, UnPoller, Proxmox) with ServiceMonitors labeled `release: kube-prometheus-stack`
- **Grafana** (standalone) configured to query the existing Prometheus

The existing Prometheus watches ServiceMonitors across all namespaces and selects those with `release: kube-prometheus-stack`. It scrapes our exporters automatically.

```text
monitoring namespace (existing):
  Prometheus Operator → manages Prometheus CRD
  Prometheus instance → scrapes ServiceMonitors with release: kube-prometheus-stack
    ↑ scrapes                    ↑ scrapes                    ↑ scrapes
    │                            │                            │
network-observability namespace (this repo):
  SNMP Exporter + SM ────────────┘                            │
  UnPoller + SM ─────────────────────────────────────────────┘
  Proxmox Exporter + SM ─────────────────────────────────────┘
  Grafana → queries Prometheus at:
    http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090
```

### Retention Limitation

The existing cluster Prometheus has **7-day retention**. The approved retention model calls for 30-day granular data. This limitation is accepted for Wave 1 — the priority is signal collection and validation. Retention will be addressed in Phase 9, potentially via Thanos, Mimir, or a dedicated long-retention Prometheus instance.

### Why This Stack

- **Standalone Grafana** avoids deploying a second Prometheus Operator (which causes CRD conflicts with the existing one).
- **Existing Prometheus** is already healthy, watching all namespaces, and has capacity for additional scrape targets.
- **SNMP Exporter** is the Prometheus project's official SNMP integration.
- **UnPoller** is the most mature Prometheus exporter for UniFi.
- **PVE Exporter** is the standard Prometheus exporter for Proxmox VE.

All components are deployed via the single ArgoCD Application. Helm charts are referenced in the Kustomize `helmCharts` field.

## Source Integration

### MikroTik CRS328 (SNMP)

- **Method:** prometheus-snmp-exporter polls the switch via SNMP.
- **Metrics:** 64-bit interface counters (ifHCInOctets, ifHCOutOctets), 32-bit and
  64-bit interface speed (ifSpeed, ifHighSpeed), operational status, system
  identity/uptime, and the MIKROTIK-MIB health sensor table (`mikrotik_health_sensor`:
  temperatures, fans, PSU voltage/current, PoE-out).
- **Labels:** Each interface metric carries `ifDescr` (port name) and `ifAlias` (port description) for human-readable identification; health sensors carry a `sensor` label.
- **Scrape interval:** 60 seconds.
- **SNMP target:** `172.18.1.13` (CRS328-24P-4S+), configured in the snmp-exporter values params.
- **Auth:** SNMP v2c, community `public` (read-only default), inlined in
  `snmp-exporter-values.yaml`. See blind spot #5 for the private-community hardening path.

### UniFi Controller / APs / WLAN Clients

- **Method:** UnPoller polls the UniFi controller REST API.
- **Metrics:** Per-client rx/tx bytes, signal strength, connection time. Per-AP traffic, client count, channel utilization. Per-site DPI stats.
- **Labels:** Client name, MAC, AP name, SSID, site.
- **Scrape interval:** 30 seconds (UnPoller internal poll), scraped by Prometheus every 30s.
- **Auth:** UniFi controller URL, username, and password from `unpoller-credentials` Secret.
- **Note:** Create a read-only local account on the UniFi controller for polling. Do not use the admin account.

### Proxmox Cluster / Nodes / VMs

- **Method:** prometheus-pve-exporter queries the Proxmox API.
- **Metrics:** Per-VM netin/netout bytes, per-node netin/netout bytes, VM status, node status.
- **Labels:** VM name, VM ID, node name, cluster.
- **Scrape interval:** 60 seconds.
- **Target:** Proxmox API endpoint configured in the ServiceMonitor params. Default: `proxmox.local:8006`.
- **Auth:** Proxmox user and password from `proxmox-credentials` Secret. Use a dedicated PVEAuditor-role API user.

## Dashboards

Three dashboards are provisioned via ConfigMap:

| Dashboard | UID | Covers |
|---|---|---|
| Switch Interface Utilization | `infra-iface-util` | Per-port bandwidth in/out, utilization % gauge (denominator = `ifHighSpeed`) |
| UniFi AP & WLAN Clients | `infra-unifi-clients` | Per-client bandwidth, client count, AP status |
| Proxmox Node & VM Network | `infra-proxmox-net` | Per-VM throughput + switch-authoritative per-host bandwidth (uplink port) |
| Network Switch & AP Hardware Health | `infra-switch-hw` | MikroTik temps/fans/PSU/PoE (SNMP) + UniFi device temp/cpu/mem/uptime (UnPoller) |

These are functional starting points — expect iteration in later phases when the full UX is built.

## Metrics Available After Deployment

### From SNMP Exporter (MikroTik)

- `ifHCInOctets` / `ifHCOutOctets` — bytes per interface (counter, use `rate()`)
- `ifOperStatus` — link up/down per interface
- `ifSpeed` — 32-bit interface speed (reads 0 above ~4Gbps; do not use as a denominator)
- `ifHighSpeed` — 64-bit interface speed in Mbps; use `ifHighSpeed * 1e6` for utilization %
- `mikrotik_health_sensor` — health table by `sensor` label (cpu/board temperature,
  fan1/2 speed, psu1/2 voltage/current, poe-out-consumption); voltage/current/power
  are deci-units (×0.1)
- `sysName` / `sysUpTime` — device identity and uptime

### From UnPoller (UniFi)

- `unpoller_client_receive_bytes_total` / `unpoller_client_transmit_bytes_total` — per-client bandwidth
- `unpoller_device_uptime_seconds` — AP uptime
- `unpoller_device_transmit_bytes_total` / `unpoller_device_receive_bytes_total` — per-AP bandwidth
- Various DPI, site, and SSID metrics

### From PVE Exporter (Proxmox)

- `pve_network_receive_bytes` / `pve_network_transmit_bytes` — per-VM network bytes
- `pve_node_info` — node status metadata (the only `pve_node_*` series present)
- **Node-level network counters are NOT exposed** — `pve_node_info_netin` /
  `pve_node_info_netout` do not exist in the live scrape (verified 2026-06-08).
  Per-host network is derived instead from the MikroTik uplink port each node
  connects to (switch-authoritative; see the Proxmox dashboard).

## Known Blind Spots

1. **MikroTik per-port only** — SNMP gives per-switch-port counters, not per-client behind a port. Client-level breakdown requires flow data (Phase 4).
2. **UniFi historical data** — UnPoller captures point-in-time snapshots. Historical data beyond Prometheus retention depends on the controller's own limited history.
3. **Proxmox aggregate counters** — VM netin/netout are cumulative since boot. Requires `rate()` for meaningful throughput. Resets on VM restart.
4. **No flow-level data** — This phase provides bandwidth/utilization only. Top talkers, destinations, app classification require the flow analytics plane (Phase 4).
5. **SNMP community in config** — The community is `public` (the switch's
   non-sensitive read-only default), inlined in `snmp-exporter-values.yaml`.
   A prior placeholder (`SNMP_COMMUNITY`) silently broke the entire scrape until
   2026-06-08; see the Wave 6 evidence. If the switch moves to a private community,
   migrate to the Secret + `--config.expand-environment-variables` pattern so the
   value is not committed to git.

## Self-Health

This repo deploys Grafana and three exporters. Prometheus and its operator are managed externally in the `monitoring` namespace.

**Deployed by this repo (network-observability):**

- Grafana has readiness/liveness probes via the Helm chart
- UnPoller and PVE Exporter have explicit liveness/readiness probes on their metrics endpoints
- SNMP Exporter has probes via the Helm chart

**External dependency (monitoring namespace):**

- Scrape target health is visible in the existing Prometheus under Status > Targets
- If the existing Prometheus in `monitoring` goes down, Grafana dashboards lose their data source — but the exporters continue running and will be scraped again when Prometheus recovers
- Existing Prometheus health is outside this repo's control — monitor it via the cluster's own monitoring stack
