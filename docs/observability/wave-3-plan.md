# Wave 3 Plan — Kubernetes Correlation

## Executive Summary

Wave 3 adds Kubernetes entity correlation on top of the Wave 2 flow plane. The
goal is to let an operator looking at a flow record (or an OpenSearch aggregation)
answer "what Kubernetes workload is this IP?" without leaving Grafana.

This is incremental enrichment, not a new data plane. Wave 2 architecture is
unchanged. Implementation is phased to validate data model decisions before
writing production manifests.

Wave 3 does not require Cilium/Hubble. It works from existing flow data and
Kubernetes API state.

---

## Problem Statement

After Wave 2, operators can see top talkers by IP, destination analysis, and
protocol/app breakdowns. What is missing:

- A flow record shows `src_ip: 10.244.3.7`. Is that a pod? Which namespace?
  Which workload?
- A top talker from the internal `10.244.0.0/16` range is unidentifiable without
  a separate kubectl lookup.
- Kubernetes node IPs (`10.x.x.x`) appear in flow records with no workload
  context attached.
- There is no dashboard surface that links "this IP in OpenSearch" to "this
  pod/service/namespace in Kubernetes".

An operator investigating a noisy workload currently has to context-switch between
Grafana and kubectl. Wave 3 closes that gap at the Grafana layer.

---

## Scope

Wave 3 includes:

- A data model and field naming convention for Kubernetes entity fields in
  flow documents
- A recommended enrichment path (see below)
- A minimal Grafana dashboard panel or variable that maps IPs to K8s entities
- Operator documentation for the correlation model
- Acceptance tests for the enrichment path

Wave 3 does NOT include:

- Cilium / Hubble deployment (this is Wave 4 / k8s-visibility plane)
- L7 visibility or HTTP tracing
- Changes to the Wave 2 flow ingestion pipeline that affect existing field types
- Service mesh integration
- Alerting rules

---

## Operator Questions Wave 3 Must Answer

An operator at Grafana should be able to answer:

1. **"This src_ip is internal — what Kubernetes workload is it?"**
   Given an IP in the `10.244.x.x` pod CIDR or a node IP, identify the
   associated pod, namespace, and workload name.

2. **"Which namespace is generating the most outbound bytes?"**
   Aggregate flow bytes by Kubernetes namespace rather than raw IP.

3. **"Is this top-talker IP a known service, a specific pod, or unknown?"**
   Distinguish: pod IP, service cluster IP, node IP, or external IP.

4. **"What pods on node X are the biggest flow contributors?"**
   Cross-reference node IP to pod list to namespace.

---

## Proposed Architecture: Kubernetes Correlation

### Enrichment target fields

Add the following fields to flow documents where the src or dst IP resolves to a
Kubernetes entity:

| Field | Type | Description |
|---|---|---|
| `src_k8s_namespace` | keyword | Namespace of the source pod/service |
| `src_k8s_workload` | keyword | Deployment/DaemonSet/StatefulSet name |
| `src_k8s_pod` | keyword | Pod name |
| `src_k8s_type` | keyword | `pod`, `node`, `service`, or `external` |
| `dst_k8s_namespace` | keyword | Namespace of the destination pod/service |
| `dst_k8s_workload` | keyword | Deployment/DaemonSet/StatefulSet name |
| `dst_k8s_pod` | keyword | Pod name |
| `dst_k8s_type` | keyword | `pod`, `node`, `service`, or `external` |

Not all flows will have these fields. Absence means the IP did not resolve to a
known Kubernetes entity at enrichment time.

### IP classification ranges

The enrichment layer needs to classify IPs into:

| Range | Classification |
|---|---|
| Pod CIDR (`10.244.0.0/16` or cluster-configured) | Pod — look up by pod IP |
| Service CIDR (`10.96.0.0/12` or cluster-configured) | Service — look up by cluster IP |
| Node IP range (known node IPs) | Node — map to k8s_node entity |
| All others | External |

---

## Enrichment Strategy Recommendation

### Options considered

**Option A — Enrich in Vector using file-based lookup tables**

Vector supports CSV enrichment tables that can be refreshed by a sidecar. A
CronJob or init-style process exports pod/service IP mappings from the Kubernetes
API to a CSV file; Vector's `get_enrichment_table_record` looks up each flow IP
against the table.

- Pros: enrichment happens at ingest time; OpenSearch documents are fully enriched;
  Grafana queries are simple aggregations.
- Cons: lookup table is eventually consistent (refresh latency = CronJob interval,
  typically 1–5 min); pod churn means some IPs are stale or missing; CSV file
  management adds operational complexity; Vector restart required to reload tables
  in some configurations.
- VRL risk: `get_enrichment_table_record` was the function that previously caused
  the GeoIP enrichment failure. Re-introducing it requires careful validation
  against the production Vector image.

**Option B — Enrich at the dashboard/query layer (Grafana variable + lookup)**

A Grafana dashboard variable queries a second datasource (a small lookup service,
or a Prometheus metric with labels) for the IP→entity mapping. The flow data in
OpenSearch is unenriched; the dashboard joins them at render time.

- Pros: no changes to the ingest pipeline; lookup is always current (real-time
  Kubernetes API); no Vector changes.
- Cons: every dashboard panel that needs K8s context must independently perform
  the join; Grafana variable-based joins are fragile and don't compose well across
  panels; aggregations (e.g. bytes by namespace) are not possible in OpenSearch
  queries without enriched fields.

**Option C — Upstream enrichment: a K8s-aware proxy or annotation service before
GoFlow2**

A service intercepts or annotates flow records before they reach GoFlow2, adding
K8s metadata based on source/dest IP. This could be a custom operator or a
well-known project (e.g. kube-enricher patterns).

- Pros: enrichment is clean and centralized upstream.
- Cons: requires a new component in the ingest path; adds a failure mode upstream
  of GoFlow2; significantly more complex to build and operate than the other
  options.

### Recommendation: Option A (Vector lookup table) with phased introduction

**Recommended path: Vector CSV enrichment tables, introduced in Phase 2.**

Rationale:

- Enriched fields in OpenSearch enable namespace/workload aggregations that are
  impossible with a query-layer join. This is the primary operator value.
- The 1–5 min staleness window is acceptable for flow analytics (flows are
  themselves 30–60s behind real time).
- The GeoIP failure was caused by invalid mmdb format on zero-byte placeholder
  files, not by the enrichment table mechanism itself. CSV tables with real data
  do not have this problem.
- Implementation is self-contained in Vector and does not require a new component.

**Mitigation for VRL/table risk:** The CronJob that exports the lookup CSV can
include a health check that only replaces the live file when the new content is
non-empty and parseable. Vector's file-based enrichment table reloads on a poll
interval without restart. Validate against the production Vector image before
activating (same process used for previous VRL fixes).

**Option B** remains a viable fallback if the CSV table approach proves operationally
difficult. The dashboard layer can do partial joins for a subset of high-value
operator panels without requiring full enrichment.

---

## Phased Implementation Plan

### Phase 1 — Design and data model (pre-implementation)

Deliverables:

- Finalize the K8s entity fields to add to the OpenSearch index template
- Confirm pod CIDR, service CIDR, and node IP ranges for the cluster
- Define the lookup table schema (columns: `ip`, `namespace`, `workload`,
  `pod`, `type`)
- Define the CronJob spec for exporting IP→entity mappings
- Define Vector enrichment table config changes required
- Gate: data model reviewed and approved before any manifest changes

Acceptance:

- Updated index template draft (field additions only, no removals)
- Lookup table schema documented
- CronJob spec sketched

### Phase 2 — Minimal enrichment path

Deliverables:

- CronJob manifest: queries Kubernetes API, exports CSV of pod and service IPs
  with namespace/workload/type fields
- ConfigMap or PVC for the lookup CSV (Vector needs a mounted path)
- Vector config update: add enrichment tables for `k8s_pods` and `k8s_services`;
  add lookup in `enrich_meta` transform for `src_ip` and `dst_ip`
- Updated OpenSearch index template with new keyword fields
- Validated against production Vector image

Acceptance:

- `vector validate` passes on production image
- At least one indexed flow document contains `src_k8s_namespace` or
  `dst_k8s_namespace` with correct value
- Existing flow fields (`src_ip`, `dst_ip`, `bytes`, `protocol`) are unchanged

### Phase 3 — Dashboards and validation

Deliverables:

- New Grafana dashboard: "K8s Flow Context" — top talkers by namespace, workload
  breakdown, node-level traffic summary
- Updated Entity Investigation dashboard: show `src_k8s_*` fields alongside
  existing flow fields
- Acceptance test procedures for Wave 3 (see Acceptance Criteria)

Acceptance:

- Dashboard loads in < 5s against live data
- Namespace aggregation shows correct grouping
- Unknown IPs gracefully show empty K8s fields (no broken panels)

### Phase 4 — Rollout and soak

Deliverables:

- Wave 3 overlay enabled in `platform/overlays/lab/kustomization.yaml`
  (if a new plane is required; otherwise ArgoCD sync of updated manifests)
- 24h soak with CronJob running and lookup tables refreshing
- Regression check of Wave 2 (flow ingest, field mapping, OpenSearch health)

Acceptance:

- Gate 3 criteria met (see go-no-go-gates.md)
- Wave 2 regression checks pass
- No Vector errors attributed to enrichment table changes

---

## Acceptance Criteria

Wave 3 is complete when:

1. Flow documents in OpenSearch contain `src_k8s_namespace` and/or
   `dst_k8s_namespace` for IP addresses in the pod/service CIDR ranges.
2. `src_k8s_type` correctly classifies pod, node, service, and external IPs.
3. A Grafana panel aggregates flow bytes by Kubernetes namespace without error.
4. The CronJob refreshes the lookup table on schedule without operator
   intervention.
5. Wave 2 ingest is unaffected: existing field types unchanged, document count
   still growing, no new Vector errors.
6. All Wave 3 operator checklist items pass.

---

## Risks and Dependencies

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Pod CIDR / service CIDR not confirmed | Medium | High — wrong ranges cause silent miss | Confirm ranges from cluster before Phase 2 |
| Vector CSV enrichment table reload causes brief outage | Low | Medium | Validate reload behavior on test pod before production apply |
| VRL reintroduction of `get_enrichment_table_record` fails on production image | Medium | High | Validate on `timberio/vector:0.45.0-distroless-static` before any changes go to cluster |
| Pod churn causes stale lookups during investigations | High | Low — flows are already delayed | Document staleness window; acceptable for operator use case |
| CronJob RBAC insufficient to read pod/service IPs | Medium | High — lookup CSV empty | Define minimal RBAC (get/list pods, services) and test in Phase 2 |
| Index template update rejected on existing indices | Low | Medium | Add new fields only; no field type changes; test on a scratch index first |

**Hard dependency:** Wave 3 does not require Cilium. Wave 4 (Hubble UI) does.
Wave 3 can proceed independently and in parallel with the Cilium migration tracked
in `k8s-lab.git`.

---

## Relationship to Wave 2 Hardening Backlog

Wave 3 begins while Wave 2 hardening items remain open. The following Wave 2
backlog items interact with Wave 3:

| Backlog item | Interaction |
|---|---|
| ISM retention verification | Complete before Wave 3 Phase 4 soak — want to confirm index rolloff before adding more fields |
| GitOps drift cleanup | Complete before Wave 3 Phase 2 — new manifests should be applied cleanly via ArgoCD, not kubectl patches |
| Top Talkers performance at 100k+ | Measure before adding K8s fields — establishes a baseline for query performance comparison |

Wave 2 hardening does not block Wave 3 Phase 1 (design work). It should be
complete before Phase 2 manifests are applied.
