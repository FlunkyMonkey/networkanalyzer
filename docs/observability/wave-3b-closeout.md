# Wave 3b Closeout — Kubernetes Flow Correlation

## Executive Summary

Wave 3b — Kubernetes IP correlation — is accepted as operational with conditions.

Flow documents in OpenSearch now carry `src_k8s_type` and `dst_k8s_type`
classifications for all flows. Pod-origin flows include namespace, workload, pod
name, and node name. Node-origin flows include the node name. CIDR-internal IPs
with no matching lookup row are classified as `internal-unknown` rather than
silently falling through to `external`. Three VRL hotfixes and one ArgoCD
GitOps fix were required during live deployment; all are committed and pushed to
`main`.

Wave 4 (K8s Visibility / Hubble) and Wave 5 (Unified UX) may begin.

---

## Implementation Summary

### Commits on main (chronological)

| Commit | Description |
|---|---|
| `1940d50` | Wave 3b Phase 2: K8s correlation manifests (initial manifest set) |
| `a773fc9` | Wave 3b Phase 2: safe-rollout hardening (suspend, change-detect, ArgoCD drift, size guard) |
| `c964c15` | Hotfix: rename `cidr_contains` to `ip_cidr_contains` in enrich\_k8s VRL |
| `8b6f688` | Hotfix: `ip_cidr_contains` fallible handling — explicit err guard (E103) |
| `9b7ce05` | Hotfix: `ignoreDifferences` for `k8s-lookup-tables` ConfigMap data |

### Phase sequence

- **Phase 1 (pre-implementation):** CIDR values and node IPs confirmed from cluster
  config, doc refinement passes completed, restart model accepted.
- **Phase 2 (manifests):** All manifests deployed via ArgoCD sync from `main`.
  Scratch-index validation passed before production template apply. Manual init
  Job triggered and verified.
- **Phase 3 (dashboards):** K8s fields visible in existing flow dashboards via
  OpenSearch query. Dedicated "K8s Flow Context" dashboard is a Wave 3b hardening
  backlog item.
- **Phase 4 (soak):** CronJob unsuspended after manual validation. Wave 2
  regression confirmed unaffected.

---

## Final Architecture Actually Deployed

### New Kubernetes resources

| Resource | Kind | Purpose |
|---|---|---|
| `k8s-ip-exporter` | ServiceAccount | Identity for the export CronJob |
| `k8s-ip-exporter-reader` | ClusterRole | get/list pods, services, nodes, replicasets across all namespaces |
| `k8s-ip-exporter-reader` | ClusterRoleBinding | Binds SA to the reader ClusterRole |
| `k8s-ip-exporter-writer` | Role (namespaced) | patch `k8s-lookup-tables` ConfigMap and `flow-collector` Deployment (named resources only) |
| `k8s-ip-exporter-writer` | RoleBinding (namespaced) | Binds SA to the writer Role |
| `k8s-ip-exporter-script` | ConfigMap | Python 3.12 export script (stdlib only, no pip deps) |
| `k8s-ip-exporter` | CronJob | Runs every 30 min; `suspend: true` by default |
| `k8s-lookup-tables` | ConfigMap | Holds pods.csv, services.csv, nodes.csv; runtime-patched by CronJob |

### Storage handoff design

ConfigMap-based handoff. `rook-ceph-block` is ReadWriteOnce — CronJob and
flow-collector pods may run on different nodes, making simultaneous PVC attachment
impossible. The CronJob PATCHes `k8s-lookup-tables` via K8s API (atomic at etcd).
The flow-collector mounts it read-only at `/etc/vector/k8s-tables/`. After each
patch, the CronJob triggers a rolling restart of flow-collector so Vector 0.45.0
reads the updated files at startup.

### Vector pipeline changes

Added `enrichment_tables` section (k8s\_pods, k8s\_services, k8s\_nodes) and a new
`enrich_k8s` transform between `enrich_meta` and the OpenSearch sink. The sink now
inputs from `enrich_k8s`. The transform implements the five-level precedence lookup:

```text
pod (k8s_pods table match)
  → service (k8s_services table match)
    → node (k8s_nodes table match)
      → internal-unknown (IP in 10.244.0.0/16 or 10.96.0.0/12, no table match)
        → external (all others)
```

### OpenSearch index template

Twelve new keyword fields added (additive only, no existing fields changed):
`src_k8s_namespace`, `src_k8s_workload`, `src_k8s_service`, `src_k8s_pod`,
`src_k8s_node`, `src_k8s_type` and the `dst_*` mirrors.

### ArgoCD Application changes

Two `ignoreDifferences` entries added to `platform/apps/network-observability.yaml`:

1. `flow-collector` Deployment — ignores `kubectl.kubernetes.io/restartedAt`
   annotation (set by CronJob rolling restart; `selfHeal: true` would otherwise
   revert it within seconds).
2. `k8s-lookup-tables` ConfigMap — ignores the `data` field (CronJob writes live
   CSV content; `selfHeal: true` would otherwise revert to placeholder headers on
   every sync cycle).

---

## Evidence of Successful Operation

### Scratch-index validation (checklist 3b.7)

Passed before production template apply. Synthetic document with all twelve K8s
keyword fields indexed without mapping exception. Scratch index deleted after
confirmation.

### Manual init Job (checklist 3b.6)

`k8s-ip-exporter-init-1` completed successfully. Logs confirmed:

- All three K8s API calls succeeded (pods, replicasets, services, nodes)
- `hostNetwork: true` pods excluded from pods.csv
- All three CSVs validated (non-empty, ≥1 data row, within 512 KB payload limit)
- `k8s-lookup-tables` ConfigMap patched
- Rolling restart of `flow-collector` triggered

### flow-collector post-restart health

`flow-collector` pod healthy after restart, Vector reporting no enrichment errors.
Enrichment tables loaded from updated ConfigMap at startup.

### Enriched flow document evidence

Pod-type flow: document contains `src_k8s_namespace`, `src_k8s_workload`,
`src_k8s_pod`, `src_k8s_node`, `src_k8s_type: "pod"`.

Node-type flow: document contains `src_k8s_node`, `src_k8s_type: "node"` for
flows originating from a known node IP (172.18.1.61–172.18.1.64).

`internal-unknown` classification: confirmed present for CIDR-internal IPs with
no matching lookup row — not silently classified as `external`.

### Wave 2 regression

Flow ingest unaffected. Existing fields (`src_ip`, `dst_ip`, `bytes`, `protocol`,
`protocol_name`) unchanged. Document count continuing to grow.

### Scheduled CronJob operation

CronJob unsuspended after manual init validation. Subsequent scheduled runs
confirmed:

- No-op exit (exit 0, "cluster state unchanged") when pod/service/node set is
  stable — no unnecessary restarts.
- Patch-and-restart triggered correctly when cluster state changes.

---

## Major Fixes During Implementation

### Fix 1 — `cidr_contains` undefined (commit `c964c15`)

**Symptom:** `flow-collector` CrashLoopBackOff immediately after Wave 3b sync.
Vector startup error: `cidr_contains` is undefined.

**Root cause:** This Vector 0.45.0 build exposes the CIDR check function as
`ip_cidr_contains`, not `cidr_contains`. The VRL docs and function name diverge
between builds.

**Fix:** Renamed both `cidr_contains(...)` calls in `enrich_k8s` to
`ip_cidr_contains(...)`. No other changes.

### Fix 2 — `ip_cidr_contains` fallible/unhandled E103 (commit `8b6f688`)

**Symptom:** `flow-collector` still CrashLoopBackOff after Fix 1.
Vector startup error: E103 unhandled fallible assignment.

**Root cause:** `ip_cidr_contains` is fallible in this build (returns
`Result<bool>`, not `bool`). Using it in a plain assignment without error handling
is rejected at compile time.

**Fix:** Expanded each CIDR check into three lines: fallible assignment, `if err
!= null { var = false }` guard, then combine with `||` into the `*_int` boolean.
Applied to both `src` and `dst` CIDR checks.

### Fix 3 — ArgoCD selfHeal reverting `k8s-lookup-tables` (commit `9b7ce05`)

**Symptom:** `flow-collector` healthy but lookup tables kept reverting to
header-only placeholder. Enriched fields absent from flow documents despite
successful CronJob run.

**Root cause:** ArgoCD `selfHeal: true` detected that the live ConfigMap `data`
field differed from the Git state (placeholder headers) and reconciled it on every
sync cycle, discarding the CronJob-written CSV content.

**Fix:** Added `ignoreDifferences` entry for `k8s-lookup-tables` ConfigMap
`.data` field in the ArgoCD Application manifest. The existing `ignoreDifferences`
for the Deployment restart annotation was already in place.

---

## Known Non-Blocking Caveats

### Dedicated K8s Flow Context dashboard not yet built

The Grafana "K8s Flow Context" dashboard referenced in checklist items 3b.15–3b.19
has not been built. K8s fields are visible via ad-hoc OpenSearch queries and the
existing Entity Investigation dashboard, but the dedicated namespace aggregation
panel is absent.

**Impact:** Operators can query K8s context but lack the purpose-built aggregation
view. Wave 4/5 UX work should include this.
**Status:** Backlog.

### CronJob runs every 30 minutes (soak cadence)

The 30-minute interval was chosen as conservative for initial soak validation. Pod
churn within a 30-minute window produces `internal-unknown` classifications until
the next CronJob run.

**Impact:** Short-lived pods (Jobs, CronJobs) may never appear as `pod`-type in
flow documents. For this homelab workload profile, this is acceptable.
**Status:** Tune to `*/5` after a stable soak period. See backlog.

### distroless Vector container has no shell

The `timberio/vector:0.45.0-distroless-static` image has no shell, package
manager, or debugging tools. `kubectl exec` is not possible during incidents.

**Impact:** VRL debugging requires either a non-distroless test image or
reproducing the config outside the cluster.
**Status:** Operational constraint. See backlog for mitigation options.

### Rolling restart causes brief enrichment gap

Each CronJob-triggered rolling restart causes ~10–30 seconds of reduced ingest
while the new pod starts up and Vector warms the enrichment tables from the
ConfigMap mount. Flow documents indexed during this window are correctly enriched
once the new pod is ready; no data is lost (GoFlow2 buffers in the shared volume).

**Impact:** Minor. At 30-minute cadence with a stable cluster (no-op majority of
runs), restarts are infrequent.
**Status:** Accepted. Documented.

### Prometheus alert for CronJob failure not yet implemented

A CronJob failure (RBAC error, API unavailable, validation failure) leaves the
previous valid lookup tables in place but produces no alert. The operator must
check `kubectl get jobs -n network-observability` manually.

**Status:** Backlog.

---

## Operator Runbook Summary

### Check enrichment table freshness

```bash
kubectl get configmap k8s-lookup-tables -n network-observability \
  -o jsonpath='{.data.pods\.csv}' | head -5
```

The `_timestamp` column (last column, Unix epoch) shows when the CronJob last
successfully wrote each row.

### Trigger a manual export run

```bash
kubectl create job --from=cronjob/k8s-ip-exporter k8s-ip-exporter-manual-$(date +%s) \
  -n network-observability
kubectl logs -n network-observability job/<name> -f
```

### Unsuspend / suspend the CronJob

```bash
# Unsuspend (start periodic operation)
kubectl patch cronjob k8s-ip-exporter -n network-observability \
  -p '{"spec":{"suspend":false}}'

# Suspend (pause without removing)
kubectl patch cronjob k8s-ip-exporter -n network-observability \
  -p '{"spec":{"suspend":true}}'
```

### Check recent CronJob history

```bash
kubectl get jobs -n network-observability --sort-by=.metadata.creationTimestamp
kubectl logs -n network-observability job/<job-name>
```

### Verify enriched fields in OpenSearch

```bash
# Port-forward to OpenSearch first, then:
curl -s 'localhost:9200/flows-*/_search?q=src_k8s_type:pod&size=1&pretty' \
  | grep -A5 "src_k8s"
```

### Roll back enrichment if Vector errors appear

```bash
# Remove the enrichment table config from vector-config.yaml in Git,
# then force ArgoCD sync:
argocd app sync network-observability --force
# Existing OpenSearch documents are unaffected — K8s fields simply absent
# on new documents until enrichment is re-enabled.
```

---

## Formal Acceptance Statement

**Wave 3b is accepted as the operational baseline.**

Criteria met:

- All five enrichment manifest types deployed and healthy
- Scratch-index validation passed (no mapping exceptions on new K8s fields)
- Manual init Job confirmed lookup tables populated with real cluster state
- `src_k8s_type` and `dst_k8s_type` present in flow documents for pod, node,
  and external classifications
- `internal-unknown` classification confirmed working (not silently `external`)
- Wave 2 ingest unaffected (existing fields unchanged, document count growing)
- Three deployment hotfixes applied and committed
- ArgoCD drift fully resolved via `ignoreDifferences`

**Wave 4 (K8s Visibility / Hubble UI) and Wave 5 (Unified UX) may begin.**

Wave 4 hard prerequisite: Cilium CNI with Hubble must be installed. If Cilium
migration has not been completed, proceed directly to Wave 5 — Wave 4 is
independent and can be completed later.
