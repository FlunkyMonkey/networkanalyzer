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
| `src_k8s_pod` | keyword | Pod name (absent for service and node IPs) |
| `src_k8s_node` | keyword | Node name the source pod runs on (absent for service/external IPs) |
| `src_k8s_type` | keyword | `pod`, `node`, `service`, or `external` |
| `dst_k8s_namespace` | keyword | Namespace of the destination pod/service |
| `dst_k8s_workload` | keyword | Deployment/DaemonSet/StatefulSet name |
| `dst_k8s_pod` | keyword | Pod name (absent for service and node IPs) |
| `dst_k8s_node` | keyword | Node name the destination pod runs on (absent for service/external IPs) |
| `dst_k8s_type` | keyword | `pod`, `node`, `service`, or `external` |

Not all flows will have these fields. Absence means the IP did not resolve to a
known Kubernetes entity at enrichment time.

### IP classification and lookup precedence

Classification is applied in order. The first matching rule wins.

| Priority | Test | Classification | Fields set |
|---|---|---|---|
| 1 | IP is in the pod CIDR range and found in pod lookup table | `pod` | namespace, workload, pod, node, type |
| 2 | IP is in the service CIDR range and found in service lookup table | `service` | namespace, workload, type (pod and node absent) |
| 3 | IP matches a known cluster node IP (from node lookup table) | `node` | node, type (namespace, workload, pod absent) |
| 4 | IP is in the pod or service CIDR range but NOT found in any lookup table | `internal-unknown` | type only — IP is internal but lookup is stale or incomplete |
| 5 | IP is outside all known CIDRs and node IPs | `external` | type only |

**Precedence rationale:** Pod lookup takes priority over service because a pod IP
is more specific and more actionable. Service lookup takes priority over node
because service cluster IPs are never node IPs. The `internal-unknown` type is
intentionally distinct from `external` — it signals a stale or incomplete lookup
rather than a genuinely external address.

**Stale lookup behavior:** If an IP was recently assigned to a now-deleted pod and
the lookup table has not yet refreshed, the IP may resolve to the old pod record.
The stale record's `_timestamp` field (added by the CronJob to each row) allows
an operator to assess data age. Staleness is expected and documented — this is
not a data quality failure for the operator use case.

**Unmatched internal IPs:** An IP in the pod CIDR that does not match any pod row
is classified as `internal-unknown` (not `external`). This preserves the signal
that something inside the cluster generated the flow even though the exact workload
is unknown. Aggregations by `src_k8s_type` will surface these as a distinct bucket.

### IP classification ranges (CIDR source of truth)

The CIDRs used for classification must be confirmed from the cluster's own
configuration, not inferred by observation. See Phase 1 gate items 3b.0 and 3b.1
in [rollout-checklists.md](rollout-checklists.md) for the authoritative lookup
procedures.

| Range | Classification |
|---|---|
| Pod CIDR (from cluster config) | Pod or internal-unknown — look up by pod IP |
| Service CIDR (from cluster config) | Service or internal-unknown — look up by cluster IP |
| Known node IPs (from `kubectl get nodes`) | Node |
| All others | External |

### Lookup table lifecycle contract

**Source of data:** A CronJob with Kubernetes API read access exports three CSV
files on each run:

- `k8s-pods.csv` — one row per running pod: `ip,namespace,workload,pod,node,_timestamp`
- `k8s-services.csv` — one row per ClusterIP service: `ip,namespace,workload,_timestamp`
- `k8s-nodes.csv` — one row per node: `ip,node,_timestamp`

The `_timestamp` column records the Unix epoch at export time. Vector does not use
this column for lookups; it is present for operator inspection of data age.

**Refresh cadence:** CronJob runs every 2 minutes. This is the staleness ceiling
for pod IP assignments. Pod churn faster than 2 minutes will produce some
`internal-unknown` classifications.

**Atomic replacement:** The CronJob writes to a `.tmp` file first, validates that
the file is non-empty and contains a parseable header, then renames it over the
live file. Vector polls for file changes on a configured interval. A write that
is interrupted mid-rename cannot produce a partial read because the rename is
atomic on the same filesystem.

**Behavior when CSV is empty or malformed:** If the CronJob produces an empty file
or a file with only a header row, the atomic rename is skipped and the previous
version remains in place. Vector continues to use the last valid table. An empty
output is treated as a transient error, not a signal to clear the lookup.

**Vector reload behavior:** Vector's file-based enrichment tables reload
automatically when the file on disk changes, without requiring a pod restart or
rollout. The reload happens within the configured scan interval (default: 30s).
During the brief window between file change and Vector reload, some flows will use
the previous table version — this is acceptable.

**Operator-visible failure mode:** If the CronJob fails (RBAC error, API
unavailable, write error), the last valid CSV remains in place. The CronJob
failure is visible via `kubectl get jobs -n network-observability` and its pod
logs. A Prometheus alert on CronJob failure is a hardening backlog item. There is
no automatic fallback — the operator must investigate the CronJob failure
directly.

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
  (including `src_k8s_node` and `dst_k8s_node` as defined above)
- Confirm pod CIDR and service CIDR from authoritative cluster sources:
  - Pod CIDR: `kubectl get node <node> -o jsonpath='{.spec.podCIDR}'` —
    this is set by the CNI and is the ground truth for per-node allocation.
    Alternatively `kubectl cluster-info dump | grep -m1 cluster-cidr`.
  - Service CIDR: `kubectl describe cm kubeadm-config -n kube-system | grep serviceSubnet`
    or, if kubeadm config is unavailable, read the kube-apiserver manifest:
    `ssh <control-plane> grep service-cluster-ip-range /etc/kubernetes/manifests/kube-apiserver.yaml`
  - Do not rely on observed IPs to infer CIDR boundaries. Use cluster
    configuration as the source of truth.
- Enumerate node IPs: `kubectl get nodes -o wide` — record the INTERNAL-IP
  column. This is the static source for node IP classification.
- Define the lookup table schema (columns: `ip`, `namespace`, `workload`,
  `pod`, `node`, `type`, `_timestamp`)
- Define the CronJob spec for exporting IP→entity mappings
- Define Vector enrichment table config changes required
- Gate: data model reviewed and approved before any manifest changes

Acceptance:

- CIDRs and node IPs documented with source commands (not inferred)
- Updated index template draft (field additions only, no removals)
- Lookup table schema documented with all columns including `_timestamp`
- CronJob spec sketched with atomic rename behavior specified

### Phase 2 — Minimal enrichment path

Deliverables:

- CronJob manifest: queries Kubernetes API, exports CSV of pod, service, and node
  IPs with namespace/workload/pod/node/type/`_timestamp` fields; uses atomic
  rename pattern
- ConfigMap or PVC for the lookup CSVs (Vector needs a mounted path)
- Vector config update: add enrichment tables for `k8s_pods`, `k8s_services`,
  and `k8s_nodes`; add precedence-ordered lookup in `enrich_meta` for `src_ip`
  and `dst_ip`
- Updated OpenSearch index template with new keyword fields (including
  `src_k8s_node`, `dst_k8s_node`, `src_k8s_type`, `dst_k8s_type`)
- Validated against production Vector image

Acceptance:

- **Scratch-index validation must pass before production template apply.** Create
  a temporary index with the new template, index a synthetic document containing
  all new K8s fields, confirm no mapping exceptions, then delete the scratch
  index. Only proceed to production template apply after this passes.
- `vector validate` passes on production image with enrichment tables configured
- At least one indexed flow document contains `src_k8s_namespace` or
  `dst_k8s_namespace` with correct value
- At least one flow document contains `src_k8s_node` or `dst_k8s_node` for a
  pod-type classification
- `src_k8s_type: "internal-unknown"` appears for an IP in the CIDR that has no
  matching pod row (confirms precedence rule 4 works)
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

- Gate 3b criteria met (see go-no-go-gates.md)
- Wave 2 regression checks pass
- No Vector errors attributed to enrichment table changes

---

## Acceptance Criteria

Wave 3 is complete when:

1. Flow documents in OpenSearch contain `src_k8s_namespace` and/or
   `dst_k8s_namespace` for IP addresses in the pod/service CIDR ranges.
2. Flow documents for pod-type IPs contain `src_k8s_node` and/or `dst_k8s_node`.
3. `src_k8s_type` correctly classifies `pod`, `node`, `service`, `internal-unknown`,
   and `external` IPs per the precedence rules defined above.
4. `internal-unknown` appears as a distinct type for CIDR-internal IPs with no
   matching lookup row — confirming it is not silently classified as `external`.
5. A Grafana panel aggregates flow bytes by Kubernetes namespace without error.
6. The CronJob refreshes the lookup tables on schedule without operator
   intervention, using atomic rename.
7. Wave 2 ingest is unaffected: existing field types unchanged, document count
   still growing, no new Vector errors.
8. All Wave 3b operator checklist items pass.

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
