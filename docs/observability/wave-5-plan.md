# Wave 5 Plan — Unified Operator UX

## Executive Summary

Wave 5 builds the unified operator UX across the three stable observability
surfaces that Waves 1–3b put in place. The deliverable is a Grafana-based
front door that gives an operator a single starting point to investigate
bandwidth issues, trace flow destinations, correlate traffic to Kubernetes
workloads, and verify that the observability platform itself is healthy.

Cilium is not required. Wave 5 works entirely from the existing data planes:
Prometheus (infra metrics), OpenSearch (flow data with K8s enrichment), and
the Kubernetes API (for CronJob health). If Wave 4 is completed later, the
Hubble surface is an additive enhancement — nothing in Wave 5 is invalidated
or replaced.

Wave 5 also closes one open Wave 3b backlog item: the K8s Flow Context
dashboard that was deferred at Wave 3b closeout.

---

## Wave 5 Objective

Deliver a unified operator UX that:

1. Provides a single homepage with the three locked investigation entry points
   (top talkers, destination analysis, WiFi client bandwidth).
2. Enables path-style investigation from a source IP to its K8s workload
   context without leaving Grafana.
3. Surfaces the K8s Flow Context (namespace breakdown, pod-to-pod flow
   volume, `internal-unknown` rate) built on the Wave 3b enrichment.
4. Provides platform self-health visibility — an operator can confirm that
   all telemetry sources are collecting and fresh.
5. Provides guided investigation playbooks for common operator scenarios.

---

## Current State Baseline

Wave 5 builds on top of these confirmed-operational components:

| Component | State | Key Facts |
|---|---|---|
| SNMP exporter | Healthy | MikroTik interface counters in Prometheus |
| UnPoller | Healthy | UniFi AP and WiFi client metrics in Prometheus |
| Proxmox exporter | Healthy | VM network metrics in Prometheus |
| Grafana | Healthy | Infra dashboards provisioned and loading |
| OpenSearch | Healthy | Flow indices growing, 14-day ISM retention active |
| GoFlow2 + Vector | Healthy | Flows ingesting from all Proxmox hosts |
| k8s-ip-exporter CronJob | Healthy | Running every 30 min with change detection |
| K8s enrichment fields | Healthy | `src_k8s_type`, `dst_k8s_type` etc. present in flow docs |

**What exists but is incomplete:**

- `platform/base/correlation-ux/` plane exists with stub ConfigMaps for
  homepage, entity investigation, platform health, and playbooks dashboards.
  These are not yet finalized or tested against real data.
- K8s Flow Context dashboard does not exist — it was deferred at Wave 3b
  closeout and is now in Wave 5 scope.
- The freshness bar logic has not been validated against a real degraded state.
- Cross-plane data links (IP → Entity Investigation, flow destination links)
  exist in design but need end-to-end testing.

---

## Scope

**Wave 5 includes:**

- K8s Flow Context dashboard (Wave 3b backlog item, now active deliverable)
- Homepage (`correlation-home`) with three locked cards and freshness bar
- Entity Investigation dashboard with K8s context for pod and service IPs
- Platform Health dashboard with scrape target status and CronJob health
- Investigation Playbooks dashboard with 6 operator playbooks
- Cross-surface navigation links and data links between all dashboards
- Freshness bar degraded-state testing (stop/restart exporter, verify banner)
- End-to-end operator validation of at least one playbook

**Wave 5 does not include:**

- Hubble UI (gated on Wave 4 / Cilium migration, currently deferred)
- Alerting rules (Prometheus alertmanager routing, PagerDuty/Slack)
- OpenSearch security hardening (TLS, authentication)
- Long-term data retention beyond the existing 14-day ISM policy
- L7 HTTP/gRPC visibility (gated on Cilium)
- Flow export from Hubble to OpenSearch (Phase 6+ correlation)
- Any changes to the Wave 3b CronJob enrichment pipeline

---

## Operator Personas and Main User Journeys

### Persona 1 — Network Operator

**Goal:** Investigate bandwidth anomalies, identify top talkers, trace unusual
traffic to a destination.

**Journey A — "Something is using too much bandwidth":**

1. Open homepage → Top Talkers card
2. Identify the suspicious source IP by bytes
3. Click the IP → Entity Investigation with IP pre-filled
4. See outbound flow table: where is it connecting? Which port? Which country?
5. Check K8s context: is this a pod? Which namespace and workload?
6. Click "View in Flow — Destination Analysis" for full breakdown over time

**Journey B — "Is our WAN link saturated right now?":**

1. Open homepage → freshness bar (confirms data is current)
2. Open Switch Interface Utilization → WAN port
3. Identify peak time and top contributing IPs from Top Talkers

### Persona 2 — Kubernetes Operator

**Goal:** Understand which K8s workloads are generating traffic, correlate
flow anomalies to specific pods or namespaces.

**Journey C — "Which namespace is generating the most egress?":**

1. Open K8s Flow Context dashboard
2. Namespace breakdown panel → sort by bytes
3. Drill into a namespace → which pods are the top talkers?
4. Cross-check `internal-unknown` rate — are there recently-started pods
   the CronJob hasn't captured yet?

**Journey D — "Why is this pod hitting the internet so much?":**

1. Start from Top Talkers → internal IP in pod CIDR
2. Click IP → Entity Investigation
3. K8s context section confirms namespace, workload, pod name
4. Outbound flow table shows destination IPs and countries
5. Check if the destinations are expected (registry pulls, telemetry, etc.)

### Persona 3 — Platform/Ops Health Monitor

**Goal:** Confirm all telemetry sources are healthy and data is fresh before
trusting any investigation.

**Journey E — "Is my observability platform actually working?":**

1. Open Platform Health dashboard
2. Check all scrape targets: SNMP, UnPoller, Proxmox, Vector — all UP?
3. Check CronJob last-success time — tables fresh?
4. Check OpenSearch document count growing
5. Check freshness bar on homepage

---

## Proposed Unified UX Architecture

### UI Surface

Grafana is the sole front door. No new frontend applications are deployed.
Grafana already has both required datasources:

| Datasource | Purpose |
|---|---|
| Prometheus (`prometheus`) | Infra metrics: interface counters, WiFi clients, VM stats, scrape health |
| OpenSearch (`opensearch-flows`) | Flow data: top talkers, destinations, K8s fields, app/port/country |

### Dashboard Plane

The `correlation-ux` Kustomize plane (`platform/base/correlation-ux/`) holds
all Wave 5 dashboards as Grafana ConfigMaps provisioned via the Grafana sidecar.
The K8s Flow Context dashboard is added to the `flow-analytics` plane since it
queries the OpenSearch flow datasource and is conceptually co-located with the
existing flow dashboards.

| Dashboard | Plane | Purpose |
|---|---|---|
| K8s Flow Context | `flow-analytics` | Namespace breakdown, pod flow volume, type breakdown |
| Network Observability — Home | `correlation-ux` | Locked homepage: top talkers, destinations, WiFi |
| Entity Investigation | `correlation-ux` | Path-style drill-down for any IP |
| Platform Health & Freshness | `correlation-ux` | Self-observability |
| Investigation Playbooks | `correlation-ux` | Guided scenarios |

### Navigation Model

Two levels of navigation:

1. **Homepage dropdowns** — Specialist Views (Entity Investigation, K8s Flow
   Context, Platform Health) and Investigation Playbooks at the top of the
   homepage.
2. **Data links on table cells** — clicking a source IP in Top Talkers opens
   Entity Investigation with the IP pre-filled; clicking a destination opens
   Flow — Destination Analysis filtered to that IP.

No new navigation infrastructure is required. Grafana dashboard variables and
data links implement all cross-surface navigation.

### Hubble Integration Placeholder

Entity Investigation includes a Hubble link that points to the Hubble UI
port-forward address. When Wave 4 is deferred:

- The link is visible in the dashboard
- It either shows a static note ("Hubble UI not deployed — see Wave 4") or
  is conditionally hidden via a Grafana variable

This preserves the Wave 4 integration path without requiring Hubble to be
present. When Wave 4 is completed, the link activates without any dashboard
changes.

---

## Recommended Implementation Approach

1. **Do not start from a blank slate.** The stub ConfigMaps in
   `platform/base/correlation-ux/` exist. Use them as the starting point;
   fill in panel definitions against real data rather than rebuilding
   from scratch.

2. **Build the K8s Flow Context dashboard first (Phase 2).** It validates
   that the Wave 3b enrichment fields are queryable from Grafana in a
   purpose-built dashboard before the homepage and entity investigation
   dashboards depend on the same fields.

3. **Test against real data, not synthetic.** Every panel must be validated
   with an actual OpenSearch query returning rows or a real Prometheus metric
   returning values. Empty panels that technically "load" are not accepted.

4. **Validate the degraded-state path.** The freshness bar must be tested
   by actually stopping an exporter and confirming the banner appears. A bar
   that shows green because it never saw a failure is not validated.

5. **Keep dashboards in version control.** All dashboard JSON is stored in
   Grafana ConfigMaps in the repo. No dashboards are created via the Grafana
   UI and left outside git. ArgoCD manages the lifecycle.

---

## Phased Plan

### Phase 1 — Baseline Audit

Verify all existing dashboards load with real data and both datasources
are healthy before building new surfaces on top of them.

Deliverables: Audit sign-off. Both datasources confirmed healthy. All Wave 1–3b
dashboards loading with real data. Wave 3b CronJob confirmed healthy.

No new files, no manifests.

### Phase 2 — K8s Flow Context Dashboard

Build the K8s Flow Context dashboard as a Grafana ConfigMap in the
`flow-analytics` plane. This closes the Wave 3b backlog item.

Panels required:

- Namespace traffic breakdown (bytes by `src_k8s_namespace` + `dst_k8s_namespace`)
- Top pod-to-pod flows (src\_k8s\_pod → dst\_k8s\_pod, bytes)
- Flow type breakdown: pod / service / node / internal-unknown / external (pie)
- Service-type flows: `src_k8s_service` + `dst_k8s_service` with namespace
- `internal-unknown` rate over time (signals stale lookup tables or high pod churn)
- Per-node flow volume (`src_k8s_node` + `dst_k8s_node`)

Deliverables: `platform/base/flow-analytics/grafana-k8s-context-dashboard.yaml`
added and kustomization.yaml updated. All six panels validated with real data.

**Gate 5a criteria met after Phase 2.**

> **Gate 5a status: ACCEPTED WITH MINOR FOLLOW-UP — 2026-04-26.**
> Seven-dashboard baseline accepted. Conditions tracked in backlog.md.
> See [go-no-go-gates.md](go-no-go-gates.md) for full verdict and evidence.

### Phase 3 — Homepage and Navigation

Finalize the homepage ConfigMap (`homepage-dashboard.yaml`) with:

- Top Talkers card (OpenSearch `terms` aggregation on `src_ip`, sort by bytes)
- Destination Analysis card (OpenSearch query filtered to selected `src_ip`)
- WiFi Client Bandwidth card (Prometheus `unpoller_client_*` metric for selected client)
- Freshness bar: exporter UP status from Prometheus, CronJob last-success time
- Specialist Views and Investigation Playbooks dropdown menus

Validate all three locked cards with real data. Test the freshness bar by
stopping and restarting each exporter.

### Phase 4 — Entity Investigation

Finalize the Entity Investigation dashboard (`investigation-dashboard.yaml`) with:

- IP variable (text input, pre-filled via data link from Top Talkers)
- K8s context section: namespace, workload, pod, node, type — queries
  OpenSearch for any flow document where `src_ip` or `dst_ip` matches
- Outbound flow table: destination IPs, ports, protocol, country, bytes
- Inbound flow table: source IPs hitting this entity
- Flow volume time series
- Links: "View in Flow — Destination Analysis" (pre-filled), "View in K8s
  Flow Context" (pre-filled to namespace), Hubble placeholder

Validate with: a pod IP (expect K8s fields), a node IP (expect K8s node),
an external IP (expect empty K8s fields, no error).

### Phase 5 — Platform Health Dashboard

Finalize the Platform Health dashboard (`health-dashboard.yaml`) with:

- Scrape targets table: SNMP, UnPoller, Proxmox, Vector — UP/DOWN from
  `up` metric in Prometheus
- Scrape duration trends (detects slow/degraded collectors)
- OpenSearch document count (from opensearch-flows datasource, `_count` query)
- Wave 3b CronJob last-success time (from `kube_job_status_completion_time`
  in kube-state-metrics, filtered to `k8s-ip-exporter-*` jobs)
- Active alert count (if Prometheus alertmanager is configured)

Validate degraded state by stopping an exporter.

### Phase 6 — Investigation Playbooks

Finalize the Playbooks dashboard (`playbooks-dashboard.yaml`) with 6 playbooks
as text panels with numbered steps and dashboard links:

1. **Investigate slow internet** — start at WAN interface util, find top talker,
   trace destinations
2. **Identify top talker** — Top Talkers → Entity Investigation → K8s context
3. **Trace a destination** — Destination Analysis for a specific dst\_ip or country
4. **K8s workload traffic** — K8s Flow Context → identify namespace → pod-level drill-down
5. **Unusual port or protocol** — Traffic Mix → filter by unexpected port or protocol
6. **Platform health check** — Platform Health → confirm all sources collecting →
   check CronJob freshness → verify doc count growing

Operator executes at least one playbook end-to-end.

### Phase 7 — Soak and Closeout

7-day soak with daily spot-checks of homepage, datasources, and Wave 3b health.
Document evidence. Write wave-5-closeout.md. Declare platform go-live.

---

## Acceptance Criteria

Wave 5 is accepted when:

- [ ] K8s Flow Context dashboard loads with real data from Wave 3b enrichment
- [ ] `internal-unknown` classification visible as a distinct bucket in the dashboard
- [ ] Homepage loads in < 5 seconds with all 3 locked cards populated
- [ ] Freshness bar correctly shows degraded state when an exporter is stopped
- [ ] Top Talkers → Entity Investigation drill-down works (IP pre-filled)
- [ ] Entity Investigation shows K8s context for a pod IP (namespace, workload, pod, node)
- [ ] Entity Investigation handles an external IP gracefully (empty K8s fields, no error)
- [ ] Platform Health dashboard shows all targets UP
- [ ] Platform Health dashboard shows the correct degraded state when a target is down
- [ ] All 6 investigation playbooks accessible and navigable
- [ ] Operator has executed at least one playbook end-to-end
- [ ] 7-day soak: no homepage failures, no datasource errors
- [ ] Wave 3b regression: CronJob healthy, enriched fields present, doc count growing
  throughout soak
- [ ] ArgoCD shows Synced, Healthy at closeout
- [ ] wave-5-closeout.md written with evidence

**Wave 4 optional — not required:** Hubble link in Entity Investigation is present
but inactive. This is the expected state until Wave 4 is completed.

---

## Risks and Dependencies

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| OpenSearch queries too slow on homepage (terms aggregation at scale) | Medium | Medium | Add `date_histogram` pre-filter to bound query scope; test at 24h, 7d, 30d windows |
| Grafana OpenSearch plugin query syntax incompatible with OpenSearch 2.x | Low | Medium | Test each query type against the live OpenSearch during Phase 1 audit |
| Wave 3b CronJob fails during Wave 5 soak | Low | Medium | K8s Flow Context dashboard goes stale; CronJob health visible in Platform Health; fix CronJob, not Wave 5 dashboards |
| Stub ConfigMaps in correlation-ux require significant rework | Medium | Low | Stub content is a starting point — rewrite panels as needed against real data |
| Freshness bar "last document" lag not queryable from Grafana | Medium | Low | Use Vector health probe as proxy; document limitation in caveats |
| Hubble link in Entity Investigation causes confusion when Wave 4 is deferred | Low | Low | Display a clear "not available" note or conditional hide; document in ops notes |

---

## Relationship to Deferred Wave 4

Wave 5 is designed to be complete without Wave 4. The following Wave 4
integration points are designed as additive enhancements, not prerequisites:

| Wave 5 element | Without Wave 4 | With Wave 4 |
|---|---|---|
| Entity Investigation Hubble link | Inactive placeholder or "not deployed" note | Links to Hubble UI port-forward for pod IP |
| Platform Health Hubble Relay target | Absent | Added to scrape targets table |
| Grafana Hubble metrics dashboard | Not deployed | Deployed as part of Wave 4 Phase 4 |
| K8s Flow Context namespace data | From Wave 3b enrichment (30-min cadence) | Same source — Hubble does not replace Wave 3b enrichment |

The K8s Flow Context dashboard uses Wave 3b enrichment data regardless of whether
Wave 4 is completed. Hubble and the Wave 3b CronJob are complementary: Wave 3b
provides enriched historical flow records in OpenSearch; Hubble provides real-time
pod flow investigation in the Hubble UI. They are not the same thing.
