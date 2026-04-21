# Wave 4 Plan — Kubernetes Network Visibility (Cilium + Hubble)

> **Status: DEFERRED**
>
> Wave 4 is blocked on Cilium CNI with Hubble being installed in the cluster.
> That migration is managed in `k8s-lab.git` and has not been scheduled.
> Wave 5 (Unified UX) is the active wave. Wave 4 planning content is preserved
> below for when the CNI migration is ready to proceed.
>
> See [backlog.md](backlog.md) — "Wave 4: Deferred" section.

---

## Executive Summary

Wave 4 adds real-time Kubernetes network visibility via Cilium's Hubble observability
layer. The deliverable for this repo is a Hubble UI deployment in `network-observability`
that shows pod-to-pod flows, namespace traffic aggregation, and service-level
connectivity — complementing the Wave 3b enrichment approach with a dedicated K8s
network investigation surface.

**Wave 4 is blocked on a hard prerequisite that is not yet met.** Cilium CNI
with Hubble must be installed and healthy in the cluster before any manifests from
this repo can be applied. The current cluster CNI state has not been confirmed and
the Cilium migration has not been scheduled. Wave 4 is deferred to backlog until
the migration is completed in `k8s-lab.git`.

Wave 5 (Unified UX) does not require Cilium and proceeds independently. If Wave 4
is completed later, Hubble integration becomes an additive enhancement on top of the
Wave 5 UX baseline.

**No implementation manifests are created for Wave 4.** Phase 1 and 2 are planning
and preflight work only. Manifest development begins in Phase 3, after the CNI
migration has been completed and Hubble Relay is confirmed reachable.

---

## Wave 4 Objective

Deploy Hubble UI as a standalone Deployment in `network-observability` that:

1. Provides real-time L3/L4 pod flow visualization for all cluster namespaces.
2. Enables per-namespace traffic breakdowns (source/destination pod, bytes, packets,
   protocol, verdict).
3. Shows service-to-service connectivity and DNS query flows within the cluster.
4. Provides the K8s-native visibility surface for cross-plane correlation in Wave 5.

Hubble UI is the only component this repo deploys. Cilium, Cilium operator,
Hubble agent, and Hubble Relay are cluster infrastructure managed in `k8s-lab.git`.

---

## Current State Assumption

Wave 3b is the stable operational baseline:

- Flow documents in OpenSearch carry `src_k8s_type` / `dst_k8s_type` for all flows.
- Pod-origin flows carry namespace, workload, pod name, and node name via the
  k8s-ip-exporter CronJob → Vector enrichment pipeline.
- The CronJob runs every 30 minutes with change-detection. No RBAC or Vector errors.
- Wave 2 ingest is unaffected. Document count growing.

Wave 4 does not modify the Wave 3b pipeline. The two approaches are complementary:
Wave 3b enriches historical flow records in OpenSearch; Wave 4 adds real-time
Hubble visibility of live pod flows. They share namespace/workload naming but
operate independently.

---

## Prerequisite

**Cilium CNI with Hubble must be installed before Wave 4 manifests are applied.**

Specifically required in the cluster (managed in `k8s-lab.git`, not this repo):

| Requirement | Check Command | Expected |
|---|---|---|
| Cilium DaemonSet healthy | `kubectl -n kube-system get ds cilium` | DESIRED = READY |
| Cilium operator healthy | `kubectl -n kube-system get deploy cilium-operator` | 1/1 Ready |
| Hubble enabled in Cilium config | `kubectl -n kube-system get cm cilium-config -o jsonpath='{.data.enable-hubble}'` | `"true"` |
| Hubble Relay healthy | `kubectl -n kube-system get deploy hubble-relay` | 1/1 Ready |
| Relay accessible cross-namespace | `kubectl run test --rm -i --image=busybox -- nc -zv hubble-relay.kube-system.svc.cluster.local 80` | Connection succeeded |

If these are not met, this wave must not proceed past Phase 1.

---

## Scope

**Wave 4 includes:**

- Phase 1 discovery: confirm current CNI, enumerate existing NetworkPolicies,
  assess migration risk, document findings.
- Phase 2 migration plan: produce a Cilium migration checklist for `k8s-lab.git`
  with CIDR validation, NetworkPolicy audit, and maintenance window requirements.
- Phase 3 Hubble UI deployment: manifests and ArgoCD config for Hubble UI in
  `network-observability` using cilium/cilium Helm chart standalone UI mode.
- Phase 4 dashboard integration: Grafana panels for Hubble Prometheus metrics,
  port-forward access runbook, link integration with the Entity Investigation
  surface.
- Phase 5 soak and closeout: stability confirmation, Wave 3b regression verification,
  formal acceptance.

**Wave 4 does not include:**

- L7 HTTP/gRPC visibility — requires CiliumNetworkPolicy or per-pod annotation.
  Tracked as backlog.
- Hubble flow export to OpenSearch (Hubble Sink → OpenSearch correlation). This is
  a Wave 5+ capability.
- Modifying or replacing the Wave 3b CronJob enrichment pipeline.
- Cilium NetworkPolicy migration for existing Calico/Flannel policies, beyond an
  audit and compatibility check.
- Changes to pod CIDR or service CIDR (Wave 3b VRL config and CronJob assumptions
  are tied to existing CIDR values).
- Any change to the `monitoring` namespace Prometheus stack.

---

## Preflight Questions — Required Before Phase 2 Begins

The answers to these questions determine migration approach and risk level. They
must be documented before any migration work begins in `k8s-lab.git`.

### 1. What CNI is currently installed?

```bash
kubectl get pods -n kube-system -o wide | grep -E 'calico|cilium|flannel|weave|canal'
kubectl get ds -n kube-system
```

Expected: Flannel (likely, given 10.244.0.0/16 pod CIDR) or Calico.
The cni-migration-and-rollout.md assumes Calico; this must be confirmed.

**If Flannel:** the migration runbook in cni-migration-and-rollout.md needs to be
adapted — the Calico removal step does not apply.

### 2. What is the current pod CIDR and service CIDR?

Wave 3b VRL is hardcoded to `10.244.0.0/16` (pod) and `10.96.0.0/12` (service).
Cilium must be installed using the same CIDRs or the Wave 3b `internal-unknown`
classification will break silently.

```bash
kubectl get node -o jsonpath='{.items[0].spec.podCIDR}'
kubectl describe cm kubeadm-config -n kube-system | grep -E 'podSubnet|serviceSubnet'
```

**If CIDRs would change during migration:** Wave 3b VRL must be updated before
migration, or the migration must be configured to preserve existing CIDRs. This
is a hard prerequisite — do not migrate with mismatched CIDRs.

### 3. Are there existing NetworkPolicies in the cluster?

```bash
kubectl get networkpolicies --all-namespaces
kubectl get globalnetworkpolicies 2>/dev/null && echo "Calico GlobalNetworkPolicies present"
kubectl get networksets 2>/dev/null && echo "Calico NetworkSets present"
```

Standard Kubernetes NetworkPolicies are compatible with Cilium — no migration needed.
Calico-specific resources (GlobalNetworkPolicy, NetworkSet, HostEndpoint) require
manual migration to CiliumNetworkPolicy equivalents before CNI removal.

**If Calico-specific policies exist:** they must be migrated or removed before
Calico is uninstalled. Removing Calico with these resources in place does not
delete them, but they will not be enforced and may confuse operators.

### 4. What cluster nodes are present and what is their current health?

```bash
kubectl get nodes -o wide
kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded
```

All nodes must be Ready and all system pods healthy before migration. Any existing
unhealthy pods should be resolved first.

### 5. Are there any pods using hostNetwork that could be affected?

```bash
kubectl get pods --all-namespaces -o json | jq '.items[] | select(.spec.hostNetwork==true) | .metadata.namespace + "/" + .metadata.name'
```

hostNetwork pods are excluded from the Wave 3b pod CSV. Cilium handles hostNetwork
pods differently — they are visible at the node identity level rather than pod
level. This is expected behavior; document the count before migration as a baseline.

### 6. What is the acceptable maintenance window?

The CNI migration requires a period of network disruption. Estimate:

- Calico/Flannel removal to Cilium ready: ~5–15 minutes (lab cluster, 4 nodes)
- Pod restart cycle (re-acquire Cilium identities): ~5–15 minutes depending on workload count
- Total disruption window: **~10–30 minutes**

Confirm this is acceptable. All workloads in `network-observability` will lose
connectivity during the window, including GoFlow2, Vector, and OpenSearch. GoFlow2
buffers flows in the shared PVC — no data is lost, but there will be a gap in
flow documents indexed during the window.

---

## Options Analysis

### Option A: Stay on Current CNI, Defer Wave 4

**Approach:** Do not migrate. Proceed to Wave 5. Wave 4 is documented as deferred.

**Tradeoffs:**

| Aspect | Assessment |
|---|---|
| Risk | None — no cluster change |
| K8s visibility | Not available. Wave 3b enrichment provides entity labeling but no real-time flow investigation surface |
| Wave 5 impact | Wave 5 can proceed without Wave 4. Cross-plane correlation is reduced but not blocked |
| Recovery path | Wave 4 can be done after Wave 5 as an independent hardening pass |

**When to choose:** Cluster has stable workloads with no acceptable maintenance
window. Migration complexity is assessed as too high for current phase.

### Option B: Migrate to Cilium with Hubble (Recommended)

**Approach:** Execute Cilium migration in `k8s-lab.git` during a maintenance window,
then deploy Hubble UI from this repo.

**Tradeoffs:**

| Aspect | Assessment |
|---|---|
| Risk | Moderate — CNI migration is the highest-risk single operation in the wave plan. Lab cluster with 4 nodes reduces blast radius |
| K8s visibility | Full L3/L4 pod flow visibility. L7 available with per-namespace configuration |
| Wave 5 impact | Enables full cross-plane correlation in Wave 5 |
| Time to value | Hubble UI deployment after migration is ~30 minutes of manifest and ArgoCD work |
| Wave 3b impact | Must verify CIDR continuity — CronJob and VRL both depend on stable CIDRs |

**Key risk:** The migration window causes 10–30 minutes of cluster network disruption.
GoFlow2/Vector/OpenSearch all lose connectivity. Flow indexing resumes automatically
after pods restart.

**When to choose:** A maintenance window is available, CIDR continuity is confirmed,
and no Calico-specific policies require migration work.

### Option C: Partial/Temporary Alternative

No viable partial alternative exists for Hubble visibility. Hubble is a Cilium-only
capability — there is no equivalent eBPF flow visibility tool that works with
Flannel or Calico at the pod identity level without CNI replacement.

The Wave 3b enrichment approach (CronJob → ConfigMap → Vector) provides K8s entity
labeling on historical flow records but does not provide real-time pod flow
investigation or the Hubble service map visualization.

**Decision:** Option B is recommended when a maintenance window is available.
Option A is acceptable if the window is not available within the Wave 4 planning
horizon.

---

## Recommendation

**Proceed with Option B (Cilium migration) after completing Phase 1 discovery.**

Rationale:

1. The lab cluster (4 nodes) has a manageable migration blast radius.
2. Wave 3b is stable. If the migration disrupts flow ingest, the disruption is
   bounded and recoverable via GoFlow2 buffering.
3. Hubble UI deployment post-migration is low-risk (namespaced Helm chart,
   standalone mode, no cluster-wide changes).
4. Wave 5 UX will be meaningfully stronger with the Hubble surface available.
5. Deferring Wave 4 post-Wave 5 means running the same migration risk later with
   more platform components running — earlier is cleaner.

The blocker is the Phase 1 preflight questions. Until the current CNI is confirmed
and CIDRs are validated, migration planning cannot begin.

---

## Phased Plan

### Phase 1 — Discovery and Migration Risk Assessment

**Owner:** Operator (cluster state queries only — no changes)
**Gate:** All preflight questions answered, findings documented

Deliverables:

- Current CNI identified and documented
- CIDR values confirmed from authoritative source (matches Wave 3b VRL assumptions)
- NetworkPolicy audit complete (count and types documented)
- hostNetwork pod inventory taken
- Maintenance window agreed
- Migration risk level assessed: low / medium / high
- Go/no-go decision: proceed to Phase 2 or defer Wave 4

No manifests. No cluster changes. No `k8s-lab.git` changes.

### Phase 2 — Cilium/Hubble Rollout Plan

**Owner:** Operator (`k8s-lab.git` repo)
**Gate:** Migration executed, Hubble Relay healthy, cross-namespace connectivity confirmed

Deliverables (in `k8s-lab.git`, not this repo):

- Cilium Helm values with Hubble enabled, CIDRs matching current cluster values
- Migration executed per cni-migration-and-rollout.md (adapted for actual CNI)
- All pods restarted and healthy post-migration
- Hubble Relay running in `kube-system`
- Cross-namespace Relay connectivity verified from `network-observability`
- Wave 3b health confirmed post-migration (CronJob running, enriched flows present)

This repo has no work in Phase 2. Wait for `k8s-lab.git` migration completion.

### Phase 3 — Hubble Data Validation and Observability Integration

**Owner:** Operator (this repo)
**Gate:** Hubble UI deployed, L3/L4 flows visible, Wave 3b unaffected

Deliverables (this repo):

- Hubble UI manifests: `platform/base/k8s-visibility/` plane enabled
- Kustomize overlay enabled: `../../base/k8s-visibility` uncommented in `platform/overlays/lab/kustomization.yaml`
- ArgoCD syncs and Hubble UI pod is Running
- Hubble UI accessible via port-forward
- Pod flows visible in Hubble UI for at least two namespaces
- DNS flows visible
- Network policy verdicts visible (allowed/denied)
- Wave 3b regression: CronJob running, enriched fields present, doc count growing

### Phase 4 — Dashboard and UX Integration

**Owner:** Operator (this repo)
**Gate:** Grafana panels for Hubble metrics loaded, cross-surface link added

Deliverables (this repo):

- Grafana dashboard for Hubble Prometheus metrics: drop rate, DNS latency,
  flow count by namespace, policy verdict breakdown
- Hubble metrics Prometheus scrape confirmed (ServiceMonitor in `kube-system`)
- Access runbook in operations-notes.md (port-forward pattern, namespace filter)
- Cross-surface entry: Entity Investigation dashboard links to Hubble port-forward
  URL for a given pod IP or namespace
- Platform Health dashboard updated to include Hubble Relay as a monitored target

### Phase 5 — Soak and Closeout

**Owner:** Operator
**Gate:** Gate 4 fully met, wave-4-closeout.md written, Wave 5 unblocked

Deliverables:

- Minimum 7-day soak with no Hubble UI CrashLoopBackOff or Relay reconnect loops
- Wave 3b still healthy across soak period (no CronJob failures, no Vector errors)
- Platform Health dashboard showing all targets UP
- wave-4-closeout.md written with evidence of successful operation
- Formal acceptance statement signed
- Wave 5 (Unified UX) declared unblocked

---

## Rollback Strategy

### Hubble UI Rollback (This Repo)

If Hubble UI causes issues in `network-observability`:

```bash
# Disable k8s-visibility plane in overlay
# Comment out ../../base/k8s-visibility in platform/overlays/lab/kustomization.yaml
# Commit and push — ArgoCD auto-prune removes Hubble UI resources
```

No impact on Wave 3b enrichment, flow ingest, or OpenSearch.

### Cilium Rollback (`k8s-lab.git`)

If Cilium migration causes persistent network issues, restore previous CNI per
the rollback plan in cni-migration-and-rollout.md:

1. `helm uninstall cilium -n kube-system`
2. Reinstall previous CNI
3. Restart all pods

Wave 3b impact during Cilium rollback: GoFlow2 loses connectivity during rollback
window. Flows resume after pod restart. No data in OpenSearch is lost.

If CIDR values changed during migration and were rolled back, Wave 3b VRL will
return to correct behavior automatically (CIDRs would be restored).

**Wave 3b is the stable baseline throughout Wave 4. Any rollback must preserve
Wave 3b functionality.**

---

## Risks and Dependencies

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| CNI is Flannel, not Calico — migration runbook needs adaptation | Medium | Medium | Phase 1 discovery confirms CNI; adapt runbook before proceeding |
| CIDR values change during Cilium migration | Low | **High** — breaks Wave 3b `internal-unknown` classification | Confirm CIDRs in Phase 1; configure Cilium to preserve existing CIDRs |
| Calico-specific NetworkPolicies require migration effort | Low | Medium | Phase 1 audit; may extend Phase 2 timeline |
| Flow ingest gap during migration window | High | Low | GoFlow2 buffers flows; no data loss; gap is bounded and expected |
| Hubble Relay not reachable cross-namespace | Low | Medium | Kustomize patch for Relay address; verify with nc before deploying Hubble UI |
| Wave 3b CronJob fails post-migration (pod identity changes) | Low | Low | RBAC is based on ServiceAccount, not pod identity; should be unaffected |
| Hubble UI Helm chart version incompatible with installed Cilium | Low | Low | Pin Hubble UI chart version to match Cilium version in `k8s-lab.git` |

---

## Acceptance Criteria

Wave 4 is accepted when:

- [ ] Cilium DaemonSet healthy on all nodes (`kubectl -n kube-system get ds cilium` shows DESIRED = READY)
- [ ] Hubble Relay running and healthy
- [ ] Hubble UI pod Running in `network-observability`
- [ ] L3/L4 flows visible in Hubble UI for at least two namespaces
- [ ] DNS flows visible in Hubble UI
- [ ] Network policy verdicts (allowed/denied) visible
- [ ] Grafana Hubble metrics dashboard loads with data
- [ ] Wave 3b regression: enriched K8s fields still present in flow documents
- [ ] Wave 3b regression: CronJob running without errors
- [ ] Wave 3b regression: OpenSearch document count still growing
- [ ] 7-day soak with no Hubble UI or Relay incidents
- [ ] wave-4-closeout.md written with evidence of successful operation
