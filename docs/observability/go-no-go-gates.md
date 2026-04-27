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

## Gate 4 — K8s Visibility Connected *(DEFERRED)*

> **Gate 4 is deferred.** Cilium CNI with Hubble has not been installed and no
> migration window has been scheduled. Wave 5 (Gate 5) is the active gate.
> Wave 4 planning content is preserved below. See [wave-4-plan.md](wave-4-plan.md)
> and the Wave 4 section in [backlog.md](backlog.md).

**Hard prerequisite:** Cilium CNI with Hubble + Relay must be installed and running in `kube-system` (managed in `k8s-lab.git`) before Phases 3–5 of Wave 4 can proceed. Wave 5 is independent of this prerequisite.

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

## Gate 5 — Unified UX Go-Live *(ACTIVE)*

**Active wave.** Wave 5 builds the unified operator UX across the existing stable
planes (Wave 1 infra, Wave 2 flow, Wave 3b K8s enrichment). Cilium is not required.

**Baseline required before Gate 5 work begins:**

- Gates 1, 2, 3, 3b all passed (all data planes healthy)
- Wave 4 deferred or not yet started

Wave 5 is phased. Three sub-gates track progress through implementation.

### Gate 5a — Phase 1–2: K8s Context Dashboard and Baseline Validated

**Verdict: ACCEPTED WITH MINOR FOLLOW-UP — 2026-04-26**

Codex review: PROCEED WITH CONDITIONS. OpenClaw visual validation: ACCEPT.
Evidence screenshots: `docs/evidence/wave5-pass3/`

Relevant commits: `79047e8` (table-first redesign), `b1c5229` (datasource URL fix),
`44e0db3` (OpenSearch field name fix), `bdb2572` (destination analysis fix),
`82a8dbc` (infra dashboard query fix), `7e4ffc4` (evidence screenshots).

**Seven-dashboard baseline:**

| Dashboard | Result |
|---|---|
| Flow — Traffic Mix | PASS |
| Flow — K8s Context | PASS |
| Flow — Top Talkers | PASS |
| Flow — Destination Analysis | PASS |
| Proxmox Node & VM Network | PASS |
| Switch Interface Utilization | MINOR FOLLOW-UP (bandwidth panels pass; % gauge shows No Data — `ifSpeed` is 0 for many interfaces; primary value remains) |
| UniFi AP & WLAN Clients | PASS |

**Conditions (non-blocking; tracked in backlog.md):**

1. Switch Interface % gauge: `ifSpeed` is 0 on many interfaces; bandwidth panels work and are the primary operator value.
2. Duplicate legacy Prometheus datasource (`Prometheus / PBFA97CFB590B2093`) remains live alongside canonical `Prometheus GitOps / uid: prometheus`.
3. GeoIP/ASN enrichment not configured.
4. High-volume unlabeled ports should be reviewed for future app-label enrichment.
5. `internal-unknown` rate should continue to be monitored and reduced over time.

**Required evidence:**

- [x] Audit of all existing Grafana dashboards complete — every dashboard loads with
  live data, no broken panels
- [x] Both Grafana datasources confirmed healthy: Prometheus and OpenSearch
- [x] K8s Flow Context dashboard deployed: namespace traffic breakdown, pod-to-pod
  flow volume, service-type flow volume, `internal-unknown` rate visible
- [x] `internal-unknown` classification rate panel shows real data (not empty)
- [x] At least one pod-type flow document visible in the K8s Flow Context dashboard
- [x] Wave 3b regression: CronJob healthy, enriched fields present, doc count growing

**Minimum pass criteria:** All existing dashboards healthy. K8s Flow Context
dashboard functional with real data from Wave 3b.

**Blockers:**

- Either datasource (Prometheus or OpenSearch) unhealthy
- Wave 3b CronJob failing (would invalidate K8s dashboard data)

### Gate 5b — Phase 3–5: Core Navigation Operational

**Verdict: ACCEPTED — 2026-04-26**

Codex review: ACCEPTED (after hotfix `e63b082`). Evidence: `docs/evidence/wave5b/`

Relevant commits: `505059b` (home dashboard), `f64df0d` (evidence screenshots),
`e63b082` (hotfix: correct CNI to Calico, K8s enrichment status),
`672aff7` (hotfix evidence screenshot).

**Delivered: Network Observability — Home landing dashboard**

- Dashboard UID: `net-obs-home`
- Dashboard title: `Network Observability — Home`
- Grafana URL: `http://netgrafana.vgriz.com/d/net-obs-home`
- ArgoCD: Synced / Healthy at `672aff7`. ConfigMap `grafana-home-dashboard` synced.
- Quick Health panel: shows `Flows Indexed — Last 1h`
- CNI status: Calico (hotfixed from incorrect Cilium reference)
- K8s enrichment status: lookup tables populated; CronJob suspended; refreshes on demand

**Validated navigation targets:**

| Target dashboard | Result |
|---|---|
| Flow — Traffic Mix | PASS |
| Flow — K8s Context | PASS |
| Flow — Top Talkers | PASS |
| Flow — Destination Analysis | PASS |
| Proxmox Node & VM Network | PASS |
| Switch Interface Utilization | PASS |
| UniFi AP & WLAN Clients | PASS |

Drill-down from Top Talkers: PASS (evidence: `drilldown-top-talkers.png`).

**Required evidence:**

- [x] Home dashboard (`net-obs-home`) loads in < 5 seconds
- [x] Navigation links to all 7 specialist dashboards validated
- [x] Drill-down from Top Talkers confirmed working
- [x] ArgoCD Synced, Healthy at closeout commit
- [ ] Embedded Card 1 (Top Talkers table) — carried to Gate 5c scope
- [ ] Embedded Card 2 (Destinations) — carried to Gate 5c scope
- [ ] Embedded Card 3 (WiFi Client Bandwidth) — carried to Gate 5c scope
- [ ] Freshness bar degraded-state test — carried to Gate 5c scope
- [ ] Entity Investigation drill-down from IP — carried to Gate 5c scope
- [ ] Platform Health dashboard with targets and degraded state — carried to Gate 5c scope

**Minimum pass criteria:** Homepage loads with all 3 cards populated. Freshness bar
correct. Top Talkers → Entity Investigation drill-down works. Platform Health accurate.

**Blockers:**

- OpenSearch datasource returning errors on homepage queries
- Any locked card empty (not just slow — consistently empty)

### Gate 5c — Phase 6–7: Playbooks, Soak, and Closeout

**Required evidence:**

- [ ] Investigation Playbooks dashboard accessible; all 6 playbooks listed
- [ ] At least one playbook followed end-to-end by the operator (verified by
  operator, not automated)
- [ ] "Investigate slow internet" playbook navigable start-to-finish
- [ ] 7-day soak: no homepage load failures, no datasource errors, no new Wave 3b
  regressions
- [ ] ArgoCD shows Synced, Healthy across soak period
- [ ] All previous wave regressions still passing at end of soak (Waves 1–3b)
- [ ] wave-5-closeout.md written with evidence of successful operation
- [ ] Wave 4 integration note confirmed: Entity Investigation Hubble link is
  present but inactive (displays correctly without Hubble deployed)

**Minimum pass criteria:** All 6 playbooks accessible. One playbook operator-verified
end-to-end. 7-day soak clean. Closeout doc written.

**Blockers:**

- Homepage consistently failing to load during soak
- Wave 3b regression during soak (enrichment fields dropping out)

**Rollback criteria:** UX dashboards are additive ConfigMaps — removing them does
not affect underlying data planes. If a dashboard causes Grafana instability, remove
the specific ConfigMap and re-sync. Full rollback to Wave 3b baseline by disabling
the `correlation-ux` overlay line.

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
