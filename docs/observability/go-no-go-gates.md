# Go/No-Go Gates

Explicit approval gates for each rollout wave. No wave proceeds without passing its gate.

**Approval owner:** Mike Beil (operator)

## Gate 0 — Preflight Clearance

**Required evidence:**

- [ ] All preflight checklist items (Wave 0) pass
- [ ] Kustomize renders 149+ objects with no errors
- [ ] All Kubernetes Secrets are created
- [ ] ArgoCD has repo access
- [ ] Cilium + Hubble Relay are confirmed running
- [ ] Network connectivity to MikroTik, UniFi, and Proxmox is verified
- [ ] Codex QA build validation passes (if Codex has run)

**Minimum pass criteria:** All items checked. Zero Critical or High findings from Codex QA.

**Blockers:**

- Cluster not accessible
- Cilium not installed
- Ceph StorageClass not available
- Any required Secret not created

**Rollback criteria:** N/A — nothing deployed yet.

---

## Gate 1 — Telemetry Plane Healthy

**Required evidence:**

- [ ] All Wave 1 checklist items pass
- [ ] Prometheus is scraping all 3 exporters (SNMP, UnPoller, Proxmox)
- [ ] Grafana is accessible with dashboards loaded
- [ ] PVCs are bound
- [ ] Regression checks R1.1–R1.5 pass
- [ ] Test procedures T1–T3 pass (interface util, WiFi clients, Proxmox VMs)

**Minimum pass criteria:** All 3 exporters UP. Grafana shows live data on all 3 infra dashboards.

**Blockers:**

- Any exporter not scraping after 10 minutes
- PVC not binding (Ceph issue)
- Grafana not loading

**Rollback criteria:** If any exporter remains DOWN after investigation, or if Prometheus itself is unhealthy, roll back Wave 1.

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

## Gate 4 — K8s Visibility Connected

**Required evidence:**

- [ ] All Wave 4 checklist items pass
- [ ] Hubble UI shows pod flows
- [ ] Namespace filtering works
- [ ] Waves 1–3 regression checks still pass (R4.1–R4.4)
- [ ] Test procedure T7 passes

**Minimum pass criteria:** Hubble UI connected to Relay. At least L3/L4 pod flows visible.

**Blockers:**

- Hubble UI cannot connect to Relay (cross-namespace networking issue)
- Cilium/Hubble not functioning at cluster level

**Rollback criteria:** If Hubble UI cannot connect after investigation, scale to 0 and proceed — K8s visibility is non-blocking for the rest of the platform.

---

## Gate 5 — Platform Go-Live

**Required evidence:**

- [ ] All Wave 5 checklist items pass
- [ ] Homepage displays locked card order with live data
- [ ] Cross-plane drill-down works (IP → Entity Investigation → OpenSearch → Hubble)
- [ ] All 6 investigation playbooks are accessible
- [ ] Platform Health dashboard shows all targets UP
- [ ] Degraded-state behavior verified (T9)
- [ ] Full regression suite passes (R1–R5 + T1–T11)
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
