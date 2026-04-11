# Wave 2 Codex QA Packet

**Scope:** Independent QA for Wave 2 (Flow Plane Infrastructure) and Wave 3 (Canary Softflowd Cutover) readiness.

**Role:** Codex validates manifests, docs, and schema — it does not deploy, approve, or modify. Findings go to the operator (Mike) and Claude Code for triage.

---

## QA Checklist

### Section 1: Build Validation

| ID | Check | Command | Pass Criteria |
|---|---|---|---|
| B1 | Full render with Wave 2 enabled | Temporarily uncomment `../../base/flow-analytics` in `platform/overlays/lab/kustomization.yaml`, then: `kustomize build --enable-helm platform/overlays/lab` | Zero errors. Object count increases from Wave 1 baseline. |
| B2 | Wave 1 object set unchanged | From B1 output, grep for Wave 1 objects (grafana, snmp-exporter, unpoller, proxmox-exporter) | All 4 Wave 1 Deployments still present |
| B3 | Wave 2 objects present | From B1 output, grep for flow-collector, opensearch | Deployment/flow-collector, StatefulSet/opensearch-0, ConfigMap/vector-config, ConfigMap/opensearch-index-template, ConfigMap/opensearch-ism-policy, ConfigMap/grafana-flow-datasource, ConfigMap/grafana-flow-dashboards |
| B4 | OpenSearch Dashboards NOT present | From B1 output, grep for opensearch-dashboards | Zero matches — OSD must not appear |
| B5 | Markdown lint | `npx markdownlint-cli2 "docs/**/*.md"` | Zero errors |
| B6 | No secrets in repo | Scan platform/ and docs/ for lines matching credential patterns (password/token/key followed by a value of 8+ chars). Exclude placeholder strings like `SNMP_COMMUNITY`, `<your-password>`. | Zero real credentials |

### Section 2: Manifest Review

| ID | Check | File | Expected |
|---|---|---|---|
| M1 | ISM retention is 14d | `platform/base/flow-analytics/opensearch-ism-policy.yaml` | `min_index_age: "14d"`, policy_id: `flow-retention-14d` |
| M2 | ISM delete action has no notification | `platform/base/flow-analytics/opensearch-ism-policy.yaml` | Delete state has only `{"delete":{}}` — no notification action |
| M3 | OpenSearch kube4 affinity | `platform/base/flow-analytics/opensearch-values.yaml` | `nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution` with kube4 weight=100, kube2 weight=50 |
| M4 | No OpenSearch Dashboards chart | `platform/base/flow-analytics/kustomization.yaml` | `helmCharts` has only `opensearch`, not `opensearch-dashboards` |
| M5 | Grafana datasource ConfigMap label | `platform/base/flow-analytics/grafana-flow-datasource.yaml` | Label `grafana_datasource: "1"` present |
| M6 | Grafana dashboard ConfigMap label | `platform/base/flow-analytics/grafana-flow-dashboards.yaml` | Label `grafana_dashboard: "1"` present |
| M7 | Grafana datasource uid | `platform/base/flow-analytics/grafana-flow-datasource.yaml` | datasource uid is `opensearch-flows` |
| M8 | Dashboard datasource uid matches | `platform/base/flow-analytics/grafana-flow-dashboards.yaml` | All panels reference uid `opensearch-flows` |
| M9 | SNMP_COMMUNITY placeholder | `platform/base/infra-telemetry/snmp-exporter-values.yaml` | Value is `SNMP_COMMUNITY` (placeholder, not real value) |
| M10 | LoadBalancer Service for flow collector | `platform/base/flow-analytics/flow-collector-deployment.yaml` | Service type is `LoadBalancer`, port 2055/UDP |
| M11 | OpenSearch security disabled | `platform/base/flow-analytics/opensearch-values.yaml` | `plugins.security.disabled: true` |
| M12 | GeoIP credentials optional | `platform/base/flow-analytics/flow-collector-deployment.yaml` | `secretKeyRef.optional: true` for geoip-credentials |

### Section 3: Dashboard JSON Validation

| ID | Check | Target | Expected |
|---|---|---|---|
| D1 | Top Talkers dashboard JSON valid | `grafana-flow-dashboards.yaml` → `flow-top-talkers.json` value | Valid JSON, uid: `flow-top-talkers` |
| D2 | Destinations dashboard JSON valid | `grafana-flow-dashboards.yaml` → `flow-destinations.json` value | Valid JSON, uid: `flow-destinations` |
| D3 | Traffic Mix dashboard JSON valid | `grafana-flow-dashboards.yaml` → `flow-traffic-mix.json` value | Valid JSON, uid: `flow-traffic-mix` |
| D4 | Dashboard datasource type correct | All panels in all 3 flow dashboards | `datasource.type: "grafana-opensearch-datasource"` |
| D5 | No Wave 1 dashboard UID collisions | Compare UIDs in flow dashboards vs infra-telemetry dashboards | No duplicate UIDs across `infra-iface-util`, `infra-unifi-clients`, `infra-proxmox-net`, `flow-top-talkers`, `flow-destinations`, `flow-traffic-mix` |
| D6 | Variable template references valid | `flow-destinations.json` templates | Variables `$src_ip`, `$dst_ip` are defined in templating section with type: textbox |
| D7 | No PPL queries in dashboard JSON | All panels across all 3 dashboards | `queryType` is `"lucene"` in every target — zero PPL targets |
| D8 | Lucene agg format correct | All table/piechart panel targets | Each target has `bucketAggs` array and `metrics` array; `timeField: "@timestamp"` |
| D9 | Query validation gate documented | `docs/observability/rollout-checklists.md` Wave 2 | Checklist includes a step to verify Grafana flow dashboards return actual data (not just datasource Test) |

### Section 4: Schema Contract Validation

| ID | Check | File | Expected |
|---|---|---|---|
| S1 | Index template covers all canonical fields | `platform/base/flow-analytics/opensearch-index-template.yaml` | Fields: `@timestamp`, `flow_start`, `flow_end`, `src_ip`, `dst_ip`, `src_port`, `dst_port`, `protocol`, `protocol_name`, `bytes`, `packets`, `tcp_flags`, `exporter`, `flow_type`, `app`, `dst_country*`, `dst_asn`, `dst_as_org`, `src_country*`, `src_asn`, `src_as_org` |
| S2 | IP fields typed as `ip` | `opensearch-index-template.yaml` | `src_ip`, `dst_ip`, `exporter` have `"type": "ip"` |
| S3 | byte/packet fields typed as `long` | `opensearch-index-template.yaml` | `bytes`, `packets` have `"type": "long"` |
| S4 | Vector field mapping matches index template | `platform/base/flow-analytics/vector-config.yaml` vs `opensearch-index-template.yaml` | Every field Vector writes exists in the index template with compatible type |
| S5 | Dashboard queries use correct field names | All dashboard panel queries | Field names match index template (e.g., `bytes`, `src_ip`, `dst_country`, `dst_as_org`) |

### Section 5: Documentation Cross-Reference

| ID | Check | Files | Expected |
|---|---|---|---|
| DR1 | ISM policy name consistency | `flow-ingest-and-cutover.md` bootstrap commands vs `opensearch-ism-policy.yaml` | Both reference `flow-retention-14d` |
| DR2 | Rollback target documented | `wave2-canary-cutover.md`, `flow-ingest-and-cutover.md` | Both reference `172.18.1.207:2055` as legacy rollback target |
| DR3 | Canary host consistent | `wave2-canary-cutover.md`, `flow-analytics-plane.md` | Both identify prox5 as canary host |
| DR4 | README index complete | `docs/observability/README.md` | `wave2-canary-cutover.md` and `wave2-codex-qa.md` listed in table |
| DR5 | Deployment plan Wave 2 consistent | `docs/observability/deployment-plan.md` Wave 2 section | Rollout plan matches new Wave 2 scope (no OSD reference) |
| DR6 | Go/No-Go Gate 2 consistent | `docs/observability/go-no-go-gates.md` | Gate 2 references flow-collector external IP check; no OSD references in critical path |
| DR7 | OpenSearch sizing rationale documented | `docs/observability/flow-analytics-plane.md` | Resource sizing section present with justification for 2Gi heap, 50Gi PVC |
| DR8 | Wave 2 rollout checklist updated | `docs/observability/rollout-checklists.md` | Wave 2 checklist has no OSD checks; Grafana flow dashboard check is present |

### Section 6: Wave 1 Regression (must pass — Wave 1 must not be broken)

| ID | Check | Command | Expected |
|---|---|---|---|
| R1 | Wave 1 kustomize still renders | `kustomize build --enable-helm platform/overlays/lab` (Wave 2 commented out — default state) | Same object count as Wave 1 baseline; no errors |
| R2 | Grafana values unchanged | `platform/base/infra-telemetry/grafana-values.yaml` | Prometheus datasource still present and default; sidecar config unchanged |
| R3 | ServiceMonitor labels intact | `platform/base/infra-telemetry/servicemonitors.yaml` | `release: kube-prometheus-stack` label still present |
| R4 | No Wave 2 resources leak into Wave 1 overlay | Wave 1-only kustomize build | No flow-collector, opensearch, or grafana-flow-* objects appear |

---

## Reporting Format

Use the standard Codex QA format from [codex-qa-sow.md](codex-qa-sow.md):

```text
ID: QA-W2-nnn
Severity: Critical / High / Medium / Low
Category: Build / Manifest / Dashboard / Schema / Documentation / Regression
Summary: one line
Evidence: command output or file:line reference
Expected: what should be true
Actual: what is true
Recommendation: (optional)
```

---

## Pass Criteria for Wave 2 Readiness

| Category | Pass Threshold |
|---|---|
| Build (B1–B6) | Zero failures |
| Manifest (M1–M12) | Zero Critical or High findings |
| Dashboard JSON (D1–D6) | All valid; zero UID collisions |
| Schema contract (S1–S5) | Zero mismatches |
| Documentation (DR1–DR8) | Zero inconsistencies in Critical fields (rollback target, policy names) |
| Wave 1 Regression (R1–R4) | All pass |

**Verdict options:**

- **PROCEED** — All critical/high pass. Operator may begin Wave 2 deployment.
- **REMEDIATE** — One or more Critical/High findings open. Claude Code fixes; Codex re-tests.
- **BLOCK** — Build fails or Wave 1 is regressed. Stop all wave activity.
