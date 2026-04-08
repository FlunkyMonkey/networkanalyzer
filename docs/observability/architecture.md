# Architecture

## Logical Planes

The platform is composed of four separate telemetry planes with a unified user experience.

### 1. Infrastructure / Telemetry Plane

Collects and stores raw infrastructure metrics from network devices.

- SNMP polling (switches, routers, APs, firewalls)
- Interface counters and utilization
- WLAN client metrics
- Proxmox VM network metrics
- Syslog ingestion

### 2. Flow Analytics Plane

Captures, enriches, and analyzes network traffic flows.

- NetFlow / sFlow / IPFIX collection
- GeoIP, ASN, DNS enrichment
- App and category classification
- Top talker analysis
- Destination and port analytics

### 3. Kubernetes Network Visibility Plane

Provides L7-aware deep visibility into Kubernetes cluster networking.

- eBPF-based packet capture
- Service mesh telemetry
- Pod-to-pod traffic mapping
- Namespace and workload traffic profiling
- DNS query visibility within clusters

### 4. Correlation / UX Plane

Unifies data from the other three planes into a coherent investigation experience.

- Centralized navigation and entity search
- Cross-plane drill-downs
- Path-style relationship visualization
- Degraded-state banners and freshness indicators
- Saved investigation playbooks

## Canonical Entity Model

All planes share a common entity naming model. Use these names consistently in code, docs, dashboards, and manifests.

### Network Asset

`wan` | `firewall` | `router` | `vlan` | `switch` | `switch_interface` | `ap` | `ssid`

### Client / Endpoint

`ip` | `mac` | `hostname` | `dns_name` | `country` | `asn` | `app` | `category`

### Compute Asset

`proxmox_host` | `vm` | `k8s_node`

### Kubernetes Workload

`cluster` | `namespace` | `workload` | `pod` | `service`

### Flow

`src` | `dst` | `protocol` | `port` | `bytes` | `packets` | `timestamps` | `geo` | `app` | `category`

### Incident Context

`alert` | `severity` | `preserved_window` | `linked_entities`

## Homepage Card Order (Locked)

1. Top talkers
2. What websites / destinations server X is connecting to
3. Bandwidth utilization of WiFi Client 7

## Default Investigation Flow (Locked)

**WAN -> work downward**

The primary investigation path starts at the WAN edge and drills down through the network hierarchy toward specific endpoints, flows, or workloads.

## Retention Model

| Tier | Purpose | Retention | Storage |
|---|---|---|---|
| Granular | Full-resolution data for dashboards and investigations | 30 days | Ceph PVC |
| Rollup | Daily aggregates for trend analysis and capacity planning | Multi-year | Ceph PVC |
| Incident | Alert-triggered preservation at full granularity | Until explicitly released | Dedicated PVC |

### Rollup Strategy

- Daily rollups are computed from granular data before it ages out.
- Rollups preserve per-entity, per-interface, and per-flow-summary aggregates.
- Multi-year history is served from rollups, not raw data.
- Rollup jobs must be idempotent and auditable.

### Incident Preservation

- When an alert fires, a preservation window captures full-granularity data around the incident.
- Preserved windows are exempt from normal retention aging.
- Preserved data remains linked to the alert context (entities, flows, timestamps).
- Preservation windows are released only by explicit operator action.

### Principles

- Raw source truth is kept in each telemetry plane's own storage.
- No silent data loss — expired data must be explicitly aged out with logging.
- Incident preservation windows override normal retention.
- Freshness and lag indicators must be visible for each data tier.
- Rollup completeness must be verified before granular data is aged out.

## Self-Observability

The platform must observe itself:

- Source freshness per collector
- Ingest lag per pipeline
- Dropped data indicators
- Stale-data banners in the UI
- Backend health and readiness endpoints
- Operator troubleshooting views
