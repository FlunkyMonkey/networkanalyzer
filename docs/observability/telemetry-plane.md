# Infrastructure Telemetry Plane

## Component Stack

| Component | Version | Purpose |
|---|---|---|
| kube-prometheus-stack | 83.3.0 | Prometheus + Grafana + Alertmanager foundation |
| prometheus-snmp-exporter | 9.13.1 | SNMP polling proxy for MikroTik CRS328 |
| unpoller | 2.39.0 | UniFi controller/AP/client metrics exporter |
| prometheus-pve-exporter | 3.5.3 | Proxmox VE node and VM metrics exporter |

### Why This Stack

- **Prometheus** is the standard for Kubernetes-native metrics. 30-day granular retention aligns with the approved retention model.
- **Grafana** provides visualization with ConfigMap-based dashboard provisioning — dashboards are version-controlled in git.
- **SNMP Exporter** is the Prometheus project's official SNMP integration. It operates as a proxy: Prometheus tells it which target to scrape.
- **UnPoller** is the most mature Prometheus exporter for UniFi. It polls the controller API and exposes per-client, per-AP, and per-site metrics.
- **PVE Exporter** is the standard Prometheus exporter for Proxmox VE. It exposes per-node and per-VM metrics via the Proxmox API.

All components are deployed via the single ArgoCD Application. Helm charts are referenced in the Kustomize `helmCharts` field — no separate Helm releases.

## Source Integration

### MikroTik CRS328 (SNMP)

- **Method:** prometheus-snmp-exporter polls the switch via SNMP.
- **Metrics:** 64-bit interface counters (ifHCInOctets, ifHCOutOctets), interface speed, operational status, system identity/uptime.
- **Labels:** Each metric carries `ifDescr` (port name) and `ifAlias` (port description) for human-readable identification.
- **Scrape interval:** 60 seconds.
- **SNMP target:** Configured in the snmp-exporter values as `mikrotik-crs328.local`. Override in lab overlay if DNS differs.
- **Auth:** SNMP community string is in the `snmp-module-config` ConfigMap. For production, move to SNMPv3 with credentials from a Secret.

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
| Switch Interface Utilization | `infra-iface-util` | Per-port bandwidth in/out, utilization % gauge |
| UniFi AP & WLAN Clients | `infra-unifi-clients` | Per-client bandwidth, client count, AP status |
| Proxmox Node & VM Network | `infra-proxmox-net` | Per-VM and per-node network throughput |

These are functional starting points — expect iteration in later phases when the full UX is built.

## Metrics Available After Deployment

### From SNMP Exporter (MikroTik)

- `ifHCInOctets` / `ifHCOutOctets` — bytes per interface (counter, use `rate()`)
- `ifOperStatus` — link up/down per interface
- `ifSpeed` — interface speed for utilization calculation
- `sysName` / `sysUpTime` — device identity and uptime

### From UnPoller (UniFi)

- `unifipoller_client_receive_bytes_total` / `unifipoller_client_transmit_bytes_total` — per-client bandwidth
- `unifipoller_device_uptime_seconds` — AP uptime
- `unifipoller_device_transmit_bytes_total` / `unifipoller_device_receive_bytes_total` — per-AP bandwidth
- Various DPI, site, and SSID metrics

### From PVE Exporter (Proxmox)

- `pve_guest_info_netin` / `pve_guest_info_netout` — per-VM network bytes
- `pve_node_info_netin` / `pve_node_info_netout` — per-node network bytes
- `pve_guest_info` — VM status metadata
- `pve_node_info` — node status metadata

## Known Blind Spots

1. **MikroTik per-port only** — SNMP gives per-switch-port counters, not per-client behind a port. Client-level breakdown requires flow data (Phase 4).
2. **UniFi historical data** — UnPoller captures point-in-time snapshots. Historical data beyond Prometheus retention depends on the controller's own limited history.
3. **Proxmox aggregate counters** — VM netin/netout are cumulative since boot. Requires `rate()` for meaningful throughput. Resets on VM restart.
4. **No flow-level data** — This phase provides bandwidth/utilization only. Top talkers, destinations, app classification require the flow analytics plane (Phase 4).
5. **SNMP community in ConfigMap** — The current SNMP module config has a placeholder community string. For production, use SNMPv3 with credentials from a Kubernetes Secret.

## Self-Health

- Prometheus, Grafana, and Alertmanager expose readiness/liveness probes via the Helm chart.
- UnPoller and PVE Exporter have explicit liveness/readiness probes on their metrics endpoints.
- SNMP Exporter has probes via the Helm chart.
- Scrape target health is visible in Prometheus under Status > Targets.
