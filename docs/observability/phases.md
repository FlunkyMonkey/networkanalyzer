# Phase Roadmap

## Phase 0 — Project Charter and Repo Contract

**Purpose:** Establish the project charter, repo conventions, and foundational documentation.

**Deliverables:**

- Repository directory structure for GitOps deployment
- CLAUDE.md with project instructions and development rules
- Architecture documentation (planes, entity model, UX priorities)
- Phase roadmap and acceptance test outline
- Placeholder manifest directories per telemetry plane

---

## Phase 1 — Architecture Decision and Source Mapping

**Purpose:** Map every lab source to its signals, collection methods, and the requirements it supports. Resolve open implementation decisions before building.

**Deliverables:**

- Source-to-signal matrix for all lab devices
- Requirements-to-signal mapping (which sources satisfy which requirements)
- Entity relationship mapping per source
- Blind spot and limitation inventory
- Open implementation decisions documented (only those needing in-cluster validation)
- Retention model finalized (30-day granular, daily rollups, multi-year history, incident windows)

---

## Phase 2 — GitOps Skeleton and ArgoCD Structure

**Purpose:** Stand up the single ArgoCD Application, Kustomize base/overlay structure, and namespace layout.

**Deliverables:**

- Single ArgoCD Application manifest for the platform namespace
- Kustomize base and overlay structure wired to ArgoCD
- Namespace definitions
- PVC and StorageClass declarations for Ceph-backed storage
- ArgoCD sync verified end-to-end (empty but functional)

---

## Phase 3 — Telemetry Plane Foundation

**Purpose:** Deploy core infrastructure monitoring — SNMP, interface utilization, WLAN clients, Proxmox VM metrics.

**Deliverables:**

- SNMP collector for MikroTik CRS328 switch interfaces
- UniFi controller integration for AP and WLAN client metrics
- Proxmox VM network metrics collection
- Time-series storage with 30-day granular retention
- Daily rollup pipeline (or schema for future rollups)
- Basic dashboards for interface and client utilization
- Health and readiness endpoints for all collectors

---

## Phase 4 — Flow Analytics Plane

**Purpose:** Deploy flow collection, enrichment, and analytics.

**Deliverables:**

- Flow collector ingesting from existing softflowd exporters on Proxmox hosts
- Firewalla flow data integration (if API/export available)
- GeoIP, ASN, and DNS enrichment pipeline
- App and category classification
- Top talker analytics
- Destination, port, and country pivots
- Flow storage with retention management

---

## Phase 5 — Kubernetes Network Visibility Plane

**Purpose:** Deploy L7-aware Kubernetes network visibility.

**Deliverables:**

- eBPF or service-mesh-based network visibility within the cluster
- Pod-to-pod traffic mapping
- Namespace and workload traffic profiling
- DNS query visibility within the cluster
- Kubernetes-native dashboards

---

## Phase 6 — Unified UX and Path-Style Correlation

**Purpose:** Build the unified investigation experience and cross-plane correlation.

**Deliverables:**

- Centralized navigation UI
- Entity search across all planes
- Cross-plane drill-down (e.g., IP -> flows -> K8s pod)
- Path-style relationship visualization
- Homepage with locked card order (top talkers, destinations, WiFi client)
- WAN-down default investigation flow
- Degraded-state banners and freshness indicators
- Saved investigation playbooks

---

## Phase 7 — Acceptance Testing, Runbooks, and Hardening

**Purpose:** Systematic acceptance testing, operator documentation, and security review.

**Deliverables:**

- All acceptance tests executed and passing
- Operator runbook for common tasks and troubleshooting
- Security analytics (anomaly detection, first-seen alerts, DNS anomalies)
- Alert routing and notification
- Incident context linking (alert -> entities -> flows)
- Performance validation (fast loading, responsive dashboards)

---

## Phase 8 — Deployment, Rollout, and Cutover

**Purpose:** Execute the ordered deployment of the full platform, verify prerequisites, and cut over live data sources.

**Deliverables:**

- Deployment readiness gate (all manifests render, lint, and dry-run clean)
- Required secrets created and verified (Grafana, UnPoller, Proxmox, GeoIP, ArgoCD repo access)
- ArgoCD repo access verified (private repo credentials registered)
- Cilium/Hubble prerequisite verified (CNI migrated, Relay reachable)
- Storage/PVC verification (Ceph StorageClass available, claims bound)
- Ordered rollout steps executed:
  - Namespace and base resources
  - Infra-telemetry plane (Prometheus, Grafana, exporters)
  - Flow-analytics plane (OpenSearch, GoFlow2, Vector)
  - K8s-visibility plane (Hubble UI)
  - Correlation-UX plane (when Phase 6 is complete)
- Cutover steps executed:
  - Softflowd exporters repointed to in-cluster collector
  - SNMP targets verified reachable
  - UniFi controller polling confirmed
  - Proxmox API access confirmed
  - OpenSearch index template and ISM policy loaded
  - Dashboards imported and verified
- Validation checkpoints at each rollout step (pods healthy, data flowing, scrape targets up)
- Rollback points documented and tested for each plane independently
- Go-live exit criteria met:
  - All planes receiving live data
  - Grafana dashboards populated
  - OpenSearch Dashboards showing flows
  - Hubble UI connected to Relay
  - No degraded-state warnings
  - Operator walkthrough completed

---

## Phase 9 — Retention, Rollups, and Optimization

**Purpose:** Finalize retention policy, rollup pipelines, and long-term storage optimization.

**Deliverables:**

- Daily rollup pipelines operational for all planes
- Multi-year retained history via rollups
- Incident-grade preserved windows fully implemented
- Storage optimization and capacity planning
- Retention aging is logged with no silent data loss
- Environment overlay templates (dev, staging, prod) if needed
- Final documentation review
