# Unified UX and Correlation

## Approach

**Grafana is the primary "front door" UI and the sole flow investigation surface.** Hubble UI remains as a specialist drill-down destination for K8s visibility, linked from Grafana dashboards. OpenSearch Dashboards is not deployed.

### Why Grafana as the Front Door

- Already deployed as part of the infra-telemetry plane
- Natively queries Prometheus (infra metrics) and OpenSearch (flow data) from a single UI
- Supports ConfigMap-provisioned dashboards — version-controlled in git
- Dashboard links, variables, and annotations provide cross-plane navigation without building a custom app
- Avoids deploying a separate frontend application for this phase

### What Each UI Owns

| UI | Owns | Accessed Via |
|---|---|---|
| **Grafana** | Homepage, investigation dashboards, flow dashboards, playbooks, health/freshness, cross-plane navigation | Primary entry point |
| **Hubble UI** | Real-time K8s pod/service flow visualization, L7 inspection | Linked from Grafana |

## Dashboard Organization

| Dashboard | UID | Tags | Purpose |
|---|---|---|---|
| Network Observability — Home | `correlation-home` | `homepage` | Locked homepage with top talkers, destinations, WiFi client bandwidth |
| Entity Investigation | `correlation-entity-investigation` | `investigation`, `specialist` | Path-style drill-down for any IP entity |
| Investigation Playbooks | `correlation-playbooks` | `playbook` | Guided investigation workflows |
| Platform Health & Freshness | `correlation-platform-health` | `health`, `specialist` | Self-observability and degraded-state detection |
| Switch Interface Utilization | `infra-iface-util` | (infra plane) | Per-port bandwidth |
| UniFi AP & WLAN Clients | `infra-unifi-clients` | (infra plane) | Per-client WiFi metrics |
| Proxmox Node & VM Network | `infra-proxmox-net` | (infra plane) | VM bandwidth |
| Flow — Top Talkers | `flow-top-talkers` | `flow-analytics` | Top source/dest IPs by bytes; total volume timeseries |
| Flow — Destination Analysis | `flow-destinations` | `flow-analytics`, `investigation` | Per-host destinations, ports, country breakdown |
| Flow — Traffic Mix | `flow-traffic-mix` | `flow-analytics` | Protocol, app, country, ASN distribution |

### Navigation Model

The homepage links to specialist dashboards via two mechanisms:

1. **Dropdown menus** at the top of the homepage:
   - "Specialist Views" — dashboards tagged `specialist`
   - "Investigation Playbooks" — dashboards tagged `playbook`
2. **Data links** on table cells (e.g., clicking a source IP in Top Talkers navigates to Entity Investigation)

## Cross-Plane Data Access

Grafana queries two datasources:

| Datasource | Name | Provides |
|---|---|---|
| Prometheus | `prometheus` (default) | Infra metrics: interface counters, WiFi clients, VM stats, scrape health |
| OpenSearch | `opensearch-flows` (UID) | Flow data: top talkers, destinations, countries, apps, ports |

The OpenSearch datasource is provisioned via a ConfigMap with the `grafana_datasource: "1"` label, automatically loaded by the Grafana sidecar.

The `grafana-opensearch-datasource` plugin is installed via the Grafana Helm values.

## Degraded-State and Freshness Indicators

The homepage includes a freshness bar at the top with:

- **Source Freshness** — checks `up` metric for infra exporters (SNMP, UnPoller, Proxmox)
- **Prometheus Scrape Health** — percentage of scrape targets currently up
- **Flow Ingest Status** — checks if the flow collector (Vector) is running

The Platform Health dashboard provides deeper self-observability:

- All scrape targets with up/down status
- Scrape duration trends (detects slow/degraded collection)
- Prometheus storage stats (WAL size, head chunks)
- Active firing alerts count

### Flow Freshness Limitation

Flow ingest freshness from OpenSearch (e.g., "last document indexed N minutes ago") is not directly queryable from Grafana without a custom exporter or Prometheus metric. This is documented as a Phase 7 hardening item.

## Path-Style Investigation

### Current Implementation

The Entity Investigation dashboard (`correlation-entity-investigation`) provides a structured path-style view:

```text
WAN → switch_interface → [entity IP] → flow destinations → pod/service
```

For a given IP, the dashboard shows:

- Infrastructure context (SNMP status, WiFi client bandwidth if applicable)
- Outbound flow table (destinations, ports, countries, apps, AS orgs)
- Inbound flow table (sources hitting this entity)
- Flow volume over time
- Links to Flow — Destination Analysis for full flow breakdown
- Links to Hubble UI for K8s workload context

### Path Visualization Limitation

True graphical path visualization (a rendered graph showing entity-to-entity relationships) is not implemented in this phase. The structured table + link approach provides the same investigative capability without building a custom graph engine. A graph visualization could be added later using:

- Grafana Node Graph panel (requires a compatible datasource)
- A custom API that builds entity relationship data from Prometheus + OpenSearch
- Integration with a graph database if entity correlation grows complex enough

## OpenSearch Datasource Plugin

The `grafana-opensearch-datasource` plugin is installed via the standalone Grafana Helm values (`grafana-values.yaml` in the infra-telemetry plane). The datasource itself is provisioned by a labeled ConfigMap in the flow-analytics plane (`grafana-flow-datasource.yaml`), picked up by the Grafana sidecar when Wave 2 is enabled.
