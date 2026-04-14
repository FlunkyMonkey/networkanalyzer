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

- [ ] All Wave 3 checklist items pass
- [ ] CronJob is running and producing a non-empty lookup CSV
- [ ] At least one flow document contains `src_k8s_namespace` or `dst_k8s_namespace`
- [ ] `src_k8s_type` and `dst_k8s_type` correctly classify pod, node, service, and external IPs
- [ ] Grafana "K8s Flow Context" dashboard loads and shows namespace-level aggregations
- [ ] CronJob has run at least 3 successful cycles (confirms scheduled refresh works)
- [ ] `vector validate` passes on the enriched config against the production image
- [ ] Wave 2 regression checks still pass (R2.1–R2.5)

**Minimum pass criteria:** Lookup table populated. At least pod and service IPs
resolving to correct namespace/workload. Wave 2 unaffected.

**Blockers:**

- CronJob RBAC insufficient (lookup CSV empty or missing fields)
- Vector errors attributable to enrichment table changes
- Index template update rejected by OpenSearch
- Existing flow fields changed or broken

**Rollback criteria:** If Vector enrichment table reintroduction causes indexing
failures or pipeline errors, remove the enrichment table config, redeploy, and
fall back to unenriched flows. Existing Wave 2 data is unaffected.

---

## Gate 4 — K8s Visibility Connected

**Hard prerequisite:** Cilium CNI with Hubble + Relay must be installed and running before this gate. If Cilium migration has not been completed, Gate 4 can be deferred — Waves 1–3 and Wave 5 are independent.

**Required evidence:**

- [ ] Cilium is running: `kubectl -n kube-system exec ds/cilium -- cilium status`
- [ ] Hubble Relay is running: `kubectl -n kube-system get deploy hubble-relay`
- [ ] All Wave 4 checklist items pass
- [ ] Hubble UI shows pod flows
- [ ] Namespace filtering works
- [ ] Waves 1–3 regression checks still pass (R4.1–R4.4)
- [ ] Test procedure T7 passes

**Minimum pass criteria:** Cilium healthy. Hubble UI connected to Relay. At least L3/L4 pod flows visible.

**Blockers:**

- Cilium not installed (migration not yet completed)
- Hubble UI cannot connect to Relay (cross-namespace networking issue)
- Cilium/Hubble not functioning at cluster level

**Rollback criteria:** If Hubble UI cannot connect after investigation, scale to 0 and proceed — K8s visibility is non-blocking for the rest of the platform.

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
