# Acceptance Tests

This document defines the acceptance test outline for the network observability platform. Tests are organized by domain and will be implemented incrementally as each phase delivers functionality.

## 1. Interface and Client Utilization

- [ ] Switch interface utilization is collected and displayed per interface
- [ ] Router interface utilization is collected and displayed
- [ ] AP interface metrics are visible
- [ ] WLAN client bandwidth utilization is tracked per client
- [ ] Proxmox VM network utilization is visible per VM
- [ ] Utilization data updates within expected polling interval
- [ ] Historical utilization is available for at least 30 days

## 2. Flow Analytics

- [ ] Top talkers are identified and ranked by volume
- [ ] Destination / website analysis is available per source
- [ ] App classification is applied to flows
- [ ] Category classification is applied to flows
- [ ] Country attribution is applied via GeoIP
- [ ] Port-based traffic breakdown is available
- [ ] ASN enrichment is present on flows
- [ ] DNS name resolution is applied to flow endpoints
- [ ] Flow data supports time-range queries

## 3. Kubernetes Visibility

- [ ] Pod-to-pod traffic is mapped within each cluster
- [ ] Namespace-level traffic aggregation is available
- [ ] Workload-level traffic profiling works
- [ ] Service-to-service communication is visible
- [ ] L7 protocol details are available (HTTP, gRPC, DNS)
- [ ] DNS queries within the cluster are visible
- [ ] Traffic is attributable to specific deployments/statefulsets

## 4. Path-Style Correlation

- [ ] Entity search returns results across all planes
- [ ] Clicking an entity navigates to a unified detail view
- [ ] Cross-plane drill-down works (e.g., IP -> flows -> K8s pod)
- [ ] Path visualization shows entity-to-entity relationships
- [ ] WAN-down investigation flow is navigable from homepage
- [ ] Homepage displays cards in locked order (top talkers, destinations, WiFi client)

## 5. Security Analytics

- [ ] Anomalous traffic patterns generate alerts
- [ ] New / first-seen destinations are flagged
- [ ] Port scan activity is detected
- [ ] Lateral movement indicators are surfaced
- [ ] DNS anomalies (tunneling, DGA) are detected
- [ ] Security playbooks are accessible from alert context

## 6. Platform Health and Self-Observability

- [ ] Each collector exposes health and readiness endpoints
- [ ] Source freshness is displayed per data source
- [ ] Ingest lag is measured and displayed per pipeline
- [ ] Dropped data events are logged and surfaced
- [ ] Stale-data banners appear when data exceeds freshness threshold
- [ ] Backend health dashboard is accessible to operators
- [ ] Operator troubleshooting view shows pipeline status

## 7. Retention, Rollups, and Incident Preservation

- [ ] Granular data is retained for at least 30 days
- [ ] Daily rollups are computed before granular data ages out
- [ ] Rollup completeness is verified before granular expiry
- [ ] Multi-year history is queryable via rollups
- [ ] Rollups preserve per-entity and per-interface aggregates
- [ ] Alert-triggered preservation window captures full-granularity data around incident
- [ ] Preserved incident data is linked to alert context (entities, flows, timestamps)
- [ ] Incident preservation windows are exempt from normal retention aging
- [ ] Preservation windows require explicit operator release
- [ ] Retention aging is logged (no silent data loss)

## 8. ArgoCD and GitOps

- [ ] Platform is managed by a single ArgoCD Application
- [ ] ArgoCD sync succeeds with no drift from repo state
- [ ] Kustomize overlays render correctly for each environment
- [ ] Namespace ownership is clear and intentional
