# Navigation Model

## Primary UI: Grafana

Grafana serves as the platform's front door. All investigation paths start here.

## Navigation Structure

```text
┌─────────────────────────────────────────────┐
│        Network Observability — Home          │
│  ┌───────────────────────────────────────┐   │
│  │ [Specialist Views ▾] [Playbooks ▾]   │   │
│  └───────────────────────────────────────┘   │
│                                               │
│  ┌─ Freshness Bar ───────────────────────┐   │
│  │ Source Health │ Scrape % │ Flow Ingest │   │
│  └───────────────────────────────────────┘   │
│                                               │
│  1. Top Talkers (table, clickable IPs)       │
│     └── click IP → Entity Investigation      │
│     └── click IP → Flow — Destination Analysis │
│                                               │
│  2. Destinations for Source IP (table)        │
│                                               │
│  3. WiFi Client Bandwidth (timeseries)       │
│                                               │
│  [Countries] [Busiest Interfaces] [Top Apps] │
└─────────────────────────────────────────────┘
         │              │              │
         ▼              ▼              ▼
 ┌──────────────┐ ┌───────────────┐ ┌──────────────┐
 │   Entity     │ │ Flow Dashboards│ │  Hubble UI   │
 │Investigation │ │ (Grafana)     │ │ (K8s flows)  │
 │  (Grafana)   │ │ top-talkers   │ │              │
 └──────────────┘ │ destinations  │ └──────────────┘
                  │ traffic-mix   │
                  └───────────────┘
```

## Drill-Down Paths

### Interface → Device → Flows

```text
Switch Interface Utilization dashboard
  → identify saturated port
    → (manual) look up device on port
      → Entity Investigation for device IP
        → Flow — Destination Analysis for full flow detail
```

### IP/Endpoint → Destinations

```text
Homepage Top Talkers
  → click source IP link
    → Entity Investigation (pre-filtered)
      → outbound destinations table
        → Flow — Destination Analysis for full flow breakdown
```

### Pod/Workload → Hubble → Flows

```text
Hubble UI (K8s visibility)
  → identify noisy pod/service
    → note pod IP
      → Entity Investigation for pod IP
        → correlate with external flow data
```

### Proxmox VM → Network → Flows

```text
Proxmox VM Network dashboard
  → identify high-bandwidth VM
    → note VM IP (from Proxmox metadata)
      → Entity Investigation for VM IP
        → flow destinations + bandwidth context
```

### WiFi Client → AP → Bandwidth → Flows

```text
UniFi Clients dashboard
  → identify high-bandwidth client
    → note client IP
      → Entity Investigation for client IP
        → WiFi context + flow destinations
```

## Link Implementation

| Link Type | Mechanism | Example |
|---|---|---|
| Homepage → Specialist | Dashboard tag dropdown | `tags: ["specialist"]` |
| Homepage → Playbook | Dashboard tag dropdown | `tags: ["playbook"]` |
| Table cell → Entity Investigation | Data link on IP field | `/d/correlation-entity-investigation?var-entity_ip=${__value.text}` |
| Table cell → Flow Destination Analysis | Data link on IP field | `/d/flow-destinations?var-src_ip=${__value.text}` |
| Any dashboard → Hubble UI | Static link | `http://hubble-ui:80` |
| Any dashboard → Home | Static link | `/d/correlation-home` |

## Variables

| Dashboard | Variable | Purpose |
|---|---|---|
| Homepage | `source_ip` | Filters the destination lookup table |
| Entity Investigation | `entity_ip` | The IP being investigated |
| Entity Investigation | `direction` | Whether entity is source or destination |

## Known Limitations

1. **No automatic entity resolution** — the operator must know or discover the IP to investigate. A future entity search feature could provide typeahead lookup across all planes.
2. **No graphical path visualization** — entity chains are represented as structured tables with links, not as a rendered graph. See [unified-ux-and-correlation.md](unified-ux-and-correlation.md) for future options.
3. **Hubble link is an internal cluster address** — works via `kubectl port-forward` or from within the cluster network. For external access, configure Ingress resources.
4. **Flow ingest freshness** — the homepage shows Prometheus scrape health and flow collector status, but not "last flow indexed N seconds ago" from OpenSearch directly.
