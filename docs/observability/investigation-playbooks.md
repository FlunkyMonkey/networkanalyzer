# Investigation Playbooks

Playbooks are accessible from the homepage dropdown or directly at the Investigation Playbooks dashboard (`/d/correlation-playbooks`).

## Available Playbooks

### Investigate: Slow Internet

**Entry point:** Homepage → WAN interface utilization + Top Talkers

| Step | Dashboard | What to Check |
|---|---|---|
| 1 | Switch Interface Utilization | Is the WAN uplink saturated? |
| 2 | Home — Top Talkers | Is one device consuming all bandwidth? |
| 3 | Entity Investigation | What destinations is the culprit connecting to? |
| 4 | UniFi Clients | Is the affected user on a weak WiFi signal? |
| 5 | Home — Countries | Unexpected international traffic? |

### Investigate: Noisy WiFi Client

**Entry point:** UniFi Clients dashboard → identify client → Entity Investigation

| Step | Dashboard | What to Check |
|---|---|---|
| 1 | UniFi Clients | Identify the noisy client by name/MAC |
| 2 | Entity Investigation | What destinations is the client hitting? |
| 3 | Entity Investigation — Apps | Streaming, backups, or bulk downloads? |
| 4 | Entity Investigation — Countries | Traffic to unexpected countries? |

### Investigate: Suspicious Outbound Country

**Entry point:** Homepage country pie chart → OpenSearch Dashboards

| Step | Dashboard | What to Check |
|---|---|---|
| 1 | Home — Countries | Which country has unexpected volume? |
| 2 | OpenSearch Dashboards | Filter: `dst_country:<country>`, identify source IPs |
| 3 | Entity Investigation | For each source, what else are they connecting to? |
| 4 | Entity Investigation — Ports | Known services (443, 80) or unusual ports? |
| 5 | Entity Investigation — ASN | Known CDN/cloud or unknown AS organization? |

### Investigate: Noisy Pod / Workload

**Entry point:** Hubble UI → identify pod → Entity Investigation

| Step | Dashboard | What to Check |
|---|---|---|
| 1 | Hubble UI | Filter by namespace, identify noisy pods |
| 2 | Entity Investigation | What external destinations from the pod IP? |
| 3 | Proxmox VM Network | Is the K8s node VM showing high network use? |
| 4 | Switch Interface Utilization | Is the node's switch port saturated? |

### Investigate: Top Talker Spike

**Entry point:** Homepage Top Talkers table → click IP → Entity Investigation

| Step | Dashboard | What to Check |
|---|---|---|
| 1 | Home — Top Talkers | Which IP spiked? Click to investigate. |
| 2 | Entity Investigation — Destinations | New destinations during the spike? |
| 3 | Entity Investigation — Timeline | When did the spike start/end? |
| 4 | Switch Interface Utilization | Did a port hit saturation? |

### Investigate: Interface Saturation

**Entry point:** Switch Interface Utilization dashboard → identify port

| Step | Dashboard | What to Check |
|---|---|---|
| 1 | Switch Interface Utilization | Which interface and what utilization %? |
| 2 | (manual) | Identify what device is on that port from switch config |
| 3 | Proxmox VM / UniFi Clients | Check bandwidth for the device type |
| 4 | Entity Investigation | Full flow breakdown for the device IP |

## Cross-Plane Navigation Pattern

All playbooks follow the same investigation pattern:

```text
Symptom (homepage/alert)
  → Identify source (infra-telemetry dashboards)
    → Correlate entity (Entity Investigation dashboard)
      → Deep dive (OpenSearch Dashboards / Hubble UI)
```

This mirrors the approved **WAN → work downward** default investigation flow.
