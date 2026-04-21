# Go/No-Go Gates

Explicit approval gates for each rollout wave. No wave proceeds without passing its gate.

**Approval owner:** Mike Beil (operator)

## Gate 0 — Preflight Clearance

**Required evidence:**

- [ ] All preflight checklist items (Wave 0) pass
- [ ] Kustomize renders ~28 objects with no errors (Wave 1 scope: namespace + Grafana + exporters)
- [ ] Kubernetes Secrets verified (grafana, unpoller, proxmox — already created)
- [ ] ArgoCD has repo access
- [ ] `rook-ceph-block` StorageClass is available
- [ ] Network connectivity to MikroTik, UniFi, and Proxmox is verified
- [ ] Codex QA build validation passes (if Codex has run)

**Note:** Cilium/Hubble are NOT required for Gate 0. They are gated at Gate 4.

**Minimum pass criteria:** All items checked. Zero Critical or High findings from Codex QA.

**Blockers:**

- Cluster not accessible
- `rook-ceph-block` StorageClass not available
- ArgoCD repo access not configured
- Any required Secret not created

**Rollback criteria:** N/A — nothing deployed yet.

---

## Gate 1 — Telemetry Plane Healthy

**Required evidence:**

- [ ] All Wave 1 checklist items pass
- [ ] Existing Prometheus (in `monitoring`) is scraping all 3 exporters (SNMP, UnPoller, Proxmox)
- [ ] Grafana is accessible with Prometheus datasource connected
- [ ] Grafana dashboards show live data
- [ ] Grafana PVC is bound
- [ ] Regression checks R1.1–R1.5 pass
- [ ] Test procedures T1–T3 pass (interface util, WiFi clients, Proxmox VMs)

**Minimum pass criteria:** All 3 exporters UP in existing Prometheus. Grafana shows live data on all 3 infra dashboards.

**Retention note:** Existing Prometheus has 7d retention. 30d objective deferred to Phase 9.

**Blockers:**

- Existing Prometheus not picking up ServiceMonitors (check `release: kube-prometheus-stack` label)
- Any exporter not scraping after 10 minutes
- Grafana PVC not binding
- Grafana cannot connect to existing Prometheus

**Rollback criteria:** If exporters don't appear in existing Prometheus targets after label verification, or if Grafana cannot connect, roll back Wave 1.

---

## Gate 2 — Flow Infrastructure Ready

**Required evidence:**

- [ ] All Wave 2 checklist items pass
- [ ] OpenSearch cluster health is green
- [ ] Index template and ISM policy are loaded
- [ ] Flow collector has an external IP
- [ ] Wave 1 regression checks still pass (R2.1–R2.5)
- [ ] Test procedure T11 passes (OpenSearch bootstrap verification)

**Minimum pass criteria:** OpenSearch green. Flow collector ready to receive. Wave 1 not regressed.

**Blockers:**

- OpenSearch fails to start (Java heap, PVC, etc.)
- MetalLB cannot assign an IP to flow-collector Service
- Wave 1 regression failure

**Rollback criteria:** If OpenSearch is not healthy after investigation, or if Wave 1 has regressed, roll back to Wave 1 state.

---

## Gate 3 — Flows Ingesting

**Required evidence:**

- [ ] All Wave 3 checklist items pass
- [ ] Canary host flows are arriving and correctly parsed
- [ ] All hosts are exporting flows
- [ ] OpenSearch document count is growing steadily
- [ ] Flow fields are correctly mapped (src_ip, dst_ip, bytes, etc.)
- [ ] GeoIP enrichment is present (if MaxMind key was configured)
- [ ] Waves 1–2 regression checks still pass (R3.1–R3.5)
- [ ] Test procedures T4–T6 pass (top talkers, destinations, ports/countries/apps)
- [ ] Test procedure T10 passes (softflowd cutover verification)

**Minimum pass criteria:** All softflowd hosts exporting. Flows indexed with correct field mapping. At least top talkers and destination analysis are functional.

**Blockers:**

- GoFlow2 not receiving flows (network/firewall issue)
- Vector indexing errors
- Corrupted flow data
- Wave 1 or 2 regression

**Rollback criteria:** If flows are not arriving from any host, or if data is corrupted, revert softflowd to original target on all hosts.

---

## Gate 3b — Kubernetes Correlation Active

**Prerequisite:** Gate 3 (Flows Ingesting) must pass first. Gate 3b can proceed
independently of the Cilium migration required for Gate 4.

**Required evidence:**

- [ ] All Wave 3b checklist items (3b.0–3b.25) pass
- [ ] Pod CIDR (`10.244.0.0/16`) and service CIDR (`10.96.0.0/12`) confirmed from
  cluster configuration (not inferred)
- [ ] Restart model accepted: Vector 0.45.0 restart-required; atomic-rename-then-restart
  pattern implemented in CronJob; no live-reload assumed
- [ ] Scratch-index validation passed before production index template was applied
- [ ] CronJob is running and producing non-empty CSV files for pods, services, and nodes
- [ ] CronJob has run at least 3 successful cycles (confirms atomic refresh works)
- [ ] hostNetwork pods excluded from k8s-pods.csv (node IPs absent from pod CSV)
- [ ] `vector validate` passes on the enriched config against the production image
- [ ] At least one flow document contains `src_k8s_namespace` or `dst_k8s_namespace`
- [ ] At least one pod-type flow document contains `src_k8s_node` or `dst_k8s_node`
- [ ] At least one service-type flow document contains `src_k8s_service` or `dst_k8s_service`
- [ ] `src_k8s_type: "internal-unknown"` is observed — confirms CIDR-internal
  IPs with no matching lookup row are not silently classified as `external`
- [ ] Grafana "K8s Flow Context" dashboard loads and shows namespace aggregations
- [ ] Wave 2 regression checks still pass (flow ingest unaffected, existing fields unchanged)

**Minimum pass criteria:** Lookup tables populated across all three entity types.
Pod IPs resolve to namespace, workload, pod name, and node name. Service IPs resolve
to namespace and service name (not workload). `internal-unknown` classification is
confirmed working. hostNetwork pods absent from pod CSV. Wave 2 unaffected.

**Blockers:**

- CronJob RBAC insufficient (lookup CSV empty or missing fields)
- Scratch-index validation failed (mapping exception on new fields)
- Vector errors attributable to enrichment table changes
- Index template update rejected on existing indices
- Existing flow fields changed, missing, or changed type

**Rollback criteria:** If Vector enrichment table reintroduction causes indexing
failures or pipeline errors, remove the enrichment table config from Vector,
redeploy, and fall back to unenriched flows. Existing Wave 2 data in OpenSearch
is unaffected — the new K8s fields will simply be absent on existing documents.

---

## Gate 4 — K8s Visibility Connected

**Hard prerequisite:** Cilium CNI with Hubble + Relay must be installed and running in `kube-system` (managed in `k8s-lab.git`) before Phases 3–5 of Wave 4 can proceed. If Cilium migration has not been completed, Gate 4 can be deferred — Waves 1–3 and Wave 5 are independent.

Wave 4 is itself phased. Phases 1–2 are planning and migration work (no manifests from this repo). Phases 3–5 are implementation and soak work.

### Gate 4a — Phase 1: Discovery Complete

**Required evidence:**

- [ ] Current CNI identified and documented (Flannel, Calico, or other)
- [ ] Pod CIDR and service CIDR confirmed from authoritative source — values match Wave 3b VRL (`10.244.0.0/16`, `10.96.0.0/12`)
- [ ] NetworkPolicy audit complete — count, types, any Calico-specific resources documented
- [ ] hostNetwork pod inventory taken
- [ ] Maintenance window agreed
- [ ] Migration risk level assessed and documented
- [ ] Go/no-go decision recorded: proceed to Phase 2 (migrate) or defer Wave 4

**Minimum pass criteria:** All preflight questions in wave-4-plan.md answered. CIDR continuity confirmed. Migration decision recorded.

**Blockers:**

- CIDR values differ from Wave 3b assumptions — must resolve before proceeding
- Calico-specific NetworkPolicies requiring migration effort with no resolution plan

**If deferred:** Document the deferral decision and proceed to Wave 5. Wave 4 can be completed after Wave 5 independently.

### Gate 4b — Phase 2: Cilium Migration Complete

**This gate is managed in `k8s-lab.git`, not this repo. Evidence is confirmed here before Phase 3 begins.**

**Required evidence:**

- [ ] Cilium DaemonSet healthy: `kubectl -n kube-system get ds cilium` shows DESIRED = READY
- [ ] Cilium operator healthy: `kubectl -n kube-system get deploy cilium-operator` shows 1/1 Ready
- [ ] Hubble Relay healthy: `kubectl -n kube-system get deploy hubble-relay` shows 1/1 Ready
- [ ] Hubble enabled in Cilium config: `kubectl -n kube-system get cm cilium-config -o jsonpath='{.data.enable-hubble}'` returns `"true"`
- [ ] Hubble Relay accessible cross-namespace: `kubectl run test --rm -i --image=busybox -- nc -zv hubble-relay.kube-system.svc.cluster.local 80` succeeds
- [ ] All cluster pods Running post-migration restart cycle
- [ ] Wave 3b health confirmed post-migration: CronJob ran successfully, enriched fields present in new flow documents
- [ ] Pod CIDR and service CIDR unchanged from pre-migration values

**Minimum pass criteria:** Cilium and Hubble Relay healthy. Cross-namespace Relay connectivity confirmed. Wave 3b unaffected.

**Blockers:**

- Any cluster node NotReady post-migration
- Hubble Relay not reachable from `network-observability` namespace
- Wave 3b CronJob failing post-migration
- CIDR values changed during migration

### Gate 4c — Phase 3–5: K8s Visibility Operational

**Required evidence:**

- [ ] Hubble UI pod Running in `network-observability`
- [ ] Hubble UI accessible via port-forward: `kubectl port-forward svc/hubble-ui 12000:80 -n network-observability`
- [ ] L3/L4 pod flows visible in Hubble UI for at least two namespaces
- [ ] DNS flows visible
- [ ] Network policy verdict (allowed/denied) visible for at least one flow
- [ ] Grafana Hubble metrics dashboard loads with live data
- [ ] Wave 3b regression: enriched K8s fields still present in new flow documents
- [ ] Wave 3b regression: CronJob running without errors across soak period
- [ ] Wave 3b regression: OpenSearch document count growing
- [ ] Minimum 7-day soak with no Hubble UI CrashLoopBackOff or Relay reconnect loops
- [ ] Waves 1–3 regression checks still pass

**Minimum pass criteria:** Hubble UI connected to Relay. L3/L4 pod flows visible. Wave 3b unaffected. 7-day soak clean.

**Blockers:**

- Hubble UI cannot connect to Relay (persistent — not transient startup)
- Wave 3b regression (enrichment fields absent or CronJob failing)

**Rollback criteria:** If Hubble UI cannot connect after investigation, scale to 0 and proceed to Wave 5 — K8s visibility is non-blocking for the rest of the platform. Wave 3b rollback is separate and not triggered by Hubble UI issues.

---

## Gate 5 — Platform Go-Live

**Required evidence:**

- [ ] All Wave 5 checklist items pass
- [ ] Homepage displays locked card order with live data
- [ ] Cross-plane drill-down works (IP → Entity Investigation → OpenSearch; Hubble if Wave 4 was completed)
- [ ] All 6 investigation playbooks are accessible
- [ ] Platform Health dashboard shows all targets UP
- [ ] Degraded-state behavior verified (T9)
- [ ] Regression suite passes for completed waves (R1–R3 + R5; R4 only if Wave 4 was completed)
- [ ] Codex QA report has zero Critical/High open findings
- [ ] Operator has completed end-to-end walkthrough of at least 1 playbook

**Minimum pass criteria:** All locked homepage cards populated. All drill-down paths functional. No Critical findings. Operator signoff.

**Blockers:**

- Homepage does not load
- Any locked card shows no data
- Critical Codex QA finding open

**Rollback criteria:** UX issues are non-destructive. Fix in place and re-verify. Full rollback only if underlying data planes have regressed.

---

## Approval Process

1. Operator executes the rollout checklist for the wave
2. Operator runs the relevant test procedures
3. Codex QA runs regression checks (if engaged)
4. Operator reviews evidence against the gate criteria
5. Operator makes the go/no-go call
6. If go: proceed to next wave
7. If no-go: investigate, remediate, re-test, re-evaluate

**No automated promotion.** Each wave requires explicit human approval before the next wave begins.
