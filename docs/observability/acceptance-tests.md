# Acceptance Tests

This document defines the acceptance test outline for the network observability platform. Tests are organized by domain and categorized by execution owner.

## Test Categories

| Category | Owner | When |
|---|---|---|
| **Build Validation** | CI / Claude Code | Every commit — `kustomize build`, `markdownlint`, secret scan |
| **Operator Execution** | Mike (operator) | During each rollout wave — see [operator-test-procedures.md](operator-test-procedures.md) |
| **QA/Regression** | Codex (QA contractor) | After each wave — see [regression-test-plan.md](regression-test-plan.md) and [codex-qa-sow.md](codex-qa-sow.md) |

## 1. Interface and Client Utilization

**Rollout wave:** Wave 1. **Test procedures:** T1, T2, T3.

- [ ] Switch interface utilization is collected and displayed per interface
- [ ] Router interface utilization is collected and displayed
- [ ] AP interface metrics are visible
- [ ] WLAN client bandwidth utilization is tracked per client
- [ ] Proxmox VM network utilization is visible per VM
- [ ] Utilization data updates within expected polling interval
- [ ] Historical utilization is available for at least 30 days

## 2. Flow Analytics

**Rollout wave:** Wave 3. **Test procedures:** T4, T5, T6, T10.

- [ ] Top talkers are identified and ranked by volume
- [ ] Destination / website analysis is available per source
- [ ] App classification is applied to flows (port-based mapping)
- [ ] Country attribution is applied via GeoIP (requires MaxMind key)
- [ ] Port-based traffic breakdown is available
- [ ] ASN enrichment is present on flows (requires MaxMind key)
- [ ] Flow data supports time-range queries
- [ ] Softflowd cutover verified — flows arriving from all Proxmox hosts

**Known Phase 4 limitations (not failures):**

- DNS name resolution is not yet applied to flow endpoints (deferred — DNS source not wired)
- Category classification beyond port-based is not available (deferred — needs Firewalla or nDPI)

## 3. Kubernetes Visibility

**Rollout wave:** Wave 4. **Test procedure:** T7.

**Prerequisite:** Cilium CNI with Hubble enabled (managed in `k8s-lab.git`).

- [ ] Hubble UI is connected to Hubble Relay
- [ ] Pod-to-pod traffic is visible
- [ ] Namespace-level filtering and aggregation works
- [ ] DNS queries within the cluster are visible (if Hubble DNS enabled)
- [ ] Service-to-service communication is visible

**Requires additional configuration for full coverage:**

- [ ] L7 protocol details (HTTP, gRPC) — requires CiliumNetworkPolicy or pod annotation
- [ ] Traffic attributable to specific deployments/statefulsets — requires L7 policy

## 4. Path-Style Correlation

**Rollout wave:** Wave 5. **Test procedure:** T8.

- [ ] Clicking an IP in Top Talkers navigates to Entity Investigation
- [ ] Entity Investigation shows outbound/inbound flow tables and flow timeline
- [ ] Cross-plane drill-down works (IP → flows → OpenSearch → Hubble UI)
- [ ] WAN-down investigation flow is navigable from homepage
- [ ] Homepage displays cards in locked order (top talkers, destinations, WiFi client)

**Known Phase 6 limitation:**

- [ ] Graphical path visualization is not implemented — uses structured tables + links instead

## 5. Security Analytics

**Rollout wave:** Phase 7 scope (framing only — full implementation deferred).

- [ ] Security analytics requirements are documented
- [ ] Investigation playbook for suspicious outbound country exists
- [ ] Flow data supports ad-hoc security queries in OpenSearch

**Deferred to future work:**

- Anomalous traffic pattern detection (alerting rules)
- New / first-seen destination flagging
- Port scan detection
- Lateral movement indicators
- DNS anomaly detection (tunneling, DGA)

## 6. Platform Health and Self-Observability

**Rollout wave:** Wave 5. **Test procedure:** T9.

- [ ] Each collector exposes health and readiness endpoints
- [ ] Source freshness is displayed on the homepage freshness bar
- [ ] Prometheus scrape health percentage is visible
- [ ] Flow ingest status is visible on the homepage
- [ ] Platform Health dashboard shows all scrape targets with UP/DOWN status
- [ ] Degraded-state behavior verified (exporter down → freshness bar shows DEGRADED)
- [ ] Operator troubleshooting view shows scrape duration and sample counts

**Known limitation:**

- OpenSearch flow ingest freshness (last-indexed timestamp) is not yet exposed as a Prometheus metric

## 7. Retention, Rollups, and Incident Preservation

**Rollout wave:** Phase 9 scope.

- [ ] Prometheus retains granular data for at least 30 days
- [ ] OpenSearch ISM policy deletes flow indices older than 30 days
- [ ] Retention aging is logged (no silent data loss)

**Deferred to Phase 9:**

- Daily rollups computed before granular data ages out
- Rollup completeness verification
- Multi-year history via rollups
- Alert-triggered incident preservation windows
- Incident data linked to alert context
- Explicit operator release for preservation windows

## 8. ArgoCD and GitOps

**Rollout wave:** Wave 0 + Wave 1. **Test procedure:** Rollout checklist items 0.4, 1.1, 1.2.

- [ ] Platform is managed by a single ArgoCD Application
- [ ] ArgoCD sync succeeds with no drift from repo state
- [ ] Kustomize overlays render correctly
- [ ] Namespace ownership is clear and intentional (`network-observability`)
