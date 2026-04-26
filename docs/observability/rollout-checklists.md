# Rollout Checklists

Operator-executable checklists for each rollout wave. Print, execute line by line, and sign off.

## Wave 0 — Preflight

**Already completed:** Namespace, grafana-admin-credentials, unpoller-credentials, proxmox-credentials.

| # | Check | Command | Expected | Pass |
|---|---|---|---|---|
| 0.1 | Kustomize renders | `kustomize build --enable-helm platform/overlays/lab \| grep '^kind:' \| wc -l` | ~28 objects (Wave 1 scope), no errors | [ ] |
| 0.2 | Markdown lint | `npx markdownlint-cli2 "docs/**/*.md"` | 0 errors | [ ] |
| 0.3 | Cluster accessible | `kubectl get nodes` | All nodes Ready | [ ] |
| 0.4 | ArgoCD repo registered | `argocd repo list \| grep networkanalyzer` | Repo listed | [ ] |
| 0.5 | StorageClass exists | `kubectl get sc rook-ceph-block` | Exists | [ ] |
| 0.6 | Namespace exists | `kubectl get ns network-observability` | Exists | [ ] |
| 0.7 | Secrets exist | `kubectl get secrets -n network-observability` | grafana-admin-credentials, unpoller-credentials, proxmox-credentials | [ ] |
| 0.8 | GeoIP secret (optional) | `kubectl get secret geoip-credentials -n network-observability` | Exists or skipped | [ ] |
| 0.9 | MikroTik reachable | `kubectl run snmp-test --rm -i --image=busybox -- nc -uzv mikrotik-crs328.local 161` | Connection succeeded | [ ] |
| 0.10 | UniFi reachable | `kubectl run unifi-test --rm -i --image=busybox -- nc -zv unifi.local 8443` | Connection succeeded | [ ] |
| 0.11 | Proxmox reachable | `kubectl run pve-test --rm -i --image=busybox -- nc -zv proxmox.local 8006` | Connection succeeded | [ ] |

**Note:** Cilium/Hubble are NOT checked here. They are a prerequisite for Wave 4 only.

**Gate:** All items pass → proceed to Wave 1. Any fail → stop, remediate, re-run.

---

## Wave 1 — Telemetry Plane

**Note:** This repo does not deploy Prometheus. The existing cluster Prometheus in `monitoring` scrapes our exporters via ServiceMonitors labeled `release: kube-prometheus-stack`.

| # | Check | Command | Expected | Pass |
|---|---|---|---|---|
| 1.1 | Apply ArgoCD app | `kubectl apply -f platform/apps/network-observability.yaml` | Application created | [ ] |
| 1.2 | ArgoCD sync status | `argocd app get network-observability` | Synced, Healthy | [ ] |
| 1.3 | All pods Running | `kubectl get pods -n network-observability` | All Running/Ready (4 pods: grafana, snmp, unpoller, proxmox) | [ ] |
| 1.4 | Existing Prometheus sees targets | Port-forward existing Prometheus (:9090 in monitoring), check `/targets` | snmp, unpoller, proxmox targets appear | [ ] |
| 1.5 | SNMP exporter scraping | Check Prometheus targets for snmp-exporter | UP, last scrape < 2m | [ ] |
| 1.6 | UnPoller scraping | Check Prometheus targets for unpoller | UP, last scrape < 2m | [ ] |
| 1.7 | Proxmox exporter scraping | Check Prometheus targets for proxmox-exporter | UP, last scrape < 2m | [ ] |
| 1.8 | Grafana accessible | `kubectl port-forward svc/grafana 3000:80 -n network-observability` | Login page loads | [ ] |
| 1.9 | Grafana datasource healthy | Grafana → Settings → Data Sources → Prometheus → Test | Connection successful | [ ] |
| 1.10 | Interface dashboard works | Navigate to Switch Interface Utilization | Shows live data | [ ] |
| 1.11 | UniFi dashboard works | Navigate to UniFi AP & WLAN Clients | Shows live data | [ ] |
| 1.12 | Proxmox dashboard works | Navigate to Proxmox Node & VM Network | Shows live data | [ ] |
| 1.13 | PVC bound | `kubectl get pvc -n network-observability` | Grafana PVC Bound | [ ] |

**Gate:** All items pass → proceed to Wave 2.

**Rollback trigger:** Grafana or any exporter fails to become healthy after 10 minutes. Existing Prometheus does not pick up ServiceMonitors (check labels).

**Rollback:** `kubectl delete -f platform/apps/network-observability.yaml && kubectl delete pvc --all -n network-observability`

---

## Wave 2 — Flow Plane Infrastructure

**Enable:** Uncomment `../../base/flow-analytics` in `platform/overlays/lab/kustomization.yaml`, commit, push. Wait for ArgoCD sync.

**Note:** OpenSearch Dashboards is NOT deployed. Grafana is the sole UI surface. The Grafana flow datasource and dashboards are provisioned via labeled ConfigMaps in the flow-analytics plane.

| # | Check | Command | Expected | Pass |
|---|---|---|---|---|
| 2.0 | Plane enabled in overlay | Verify flow-analytics line is uncommented in lab kustomization.yaml | Uncommented | [ ] |
| 2.1 | OpenSearch Running | `kubectl get pods -n network-observability -l app.kubernetes.io/name=opensearch` | 1/1 Running | [ ] |
| 2.2 | OpenSearch health | Port-forward 9200, `curl localhost:9200/_cluster/health` | `"status":"green"` | [ ] |
| 2.3 | OpenSearch node placement | `kubectl get pod opensearch-0 -n network-observability -o jsonpath='{.spec.nodeName}'` | kube4 (preferred) or kube2 (fallback) | [ ] |
| 2.4 | Flow collector Running | `kubectl get pods -n network-observability -l app.kubernetes.io/name=flow-collector` | 2/2 Running (goflow2 + vector) | [ ] |
| 2.5 | Collector external IP | `kubectl get svc flow-collector -n network-observability` | EXTERNAL-IP assigned by MetalLB | [ ] |
| 2.6 | Index template loaded | Port-forward 9200, `curl localhost:9200/_index_template/flows` | Template exists (not 404) | [ ] |
| 2.7 | ISM policy loaded | `curl localhost:9200/_plugins/_ism/policies/flow-retention-14d` | Policy exists, min_index_age: 14d | [ ] |
| 2.8 | Grafana datasource loaded | Grafana → Configuration → Data Sources | "OpenSearch Flows" datasource present | [ ] |
| 2.9 | Grafana datasource healthy | Grafana → Data Sources → OpenSearch Flows → Test | "Data source is working" | [ ] |
| 2.10 | Flow dashboards visible | Grafana → Dashboards → Browse | flow-top-talkers, flow-destinations, flow-traffic-mix listed | [ ] |
| 2.11 | Vector healthy | `kubectl exec deploy/flow-collector -c vector -- curl -s localhost:8686/health` | HTTP 200 | [ ] |
| 2.12 | Wave 1 still healthy | Re-run checks 1.3–1.7 | All still passing | [ ] |
| 2.13 | **Query validation** — flow dashboards return data after Wave 3 canary | After prox5 cutover: open Flow — Top Talkers, confirm table rows are populated | At least one src_ip row visible — datasource "Test" passing is not sufficient; actual query data required | [ ] |

**Gate:** All items pass → proceed to Wave 3.

**Rollback trigger:** OpenSearch PVC fails to bind. Flow collector cannot start. MetalLB fails to assign IP. Grafana datasource cannot connect to OpenSearch.

**Rollback:** Re-comment `../../base/flow-analytics` in the lab overlay, commit, push. ArgoCD auto-prune removes flow resources. Grafana loses the OpenSearch datasource and flow dashboards automatically (their ConfigMaps are pruned).

---

## Wave 3 — Softflowd Canary Cutover

**Canary host:** prox5 (already exporting NetFlow v9)
**Legacy rollback target:** `172.18.1.207:2055/UDP`
**Full cutover runbook:** See [wave2-canary-cutover.md](wave2-canary-cutover.md)

| # | Check | Command | Expected | Pass |
|---|---|---|---|---|
| 3.1 | Record all softflowd targets | `ssh <each-host> cat /etc/default/softflowd` | All documented as `172.18.1.207:2055` | [ ] |
| 3.2 | Canary: repoint prox5 | Update prox5 softflowd to FLOW_COLLECTOR_IP:2055, restart | softflowd active/running | [ ] |
| 3.3 | GoFlow2 receiving | `kubectl exec deploy/flow-collector -c goflow2 -- tail -3 /flows/flows.jsonl` | JSON records appearing (exporter field = prox5 IP) | [ ] |
| 3.4 | OpenSearch indexing | Port-forward 9200, `curl localhost:9200/flows-*/_count` | Count > 0 and growing | [ ] |
| 3.5 | Flow fields correct | `curl 'localhost:9200/flows-*/_search?size=1&pretty'` | src_ip, dst_ip, bytes, protocol_name, exporter present | [ ] |
| 3.6 | GeoIP enrichment (if configured) | Check `dst_country` in a flow doc | Present if MaxMind key was set | [ ] |
| 3.7 | Canary soak: 15 minutes | Monitor doc count growth for 15 min | Steady increase, no Vector errors | [ ] |
| 3.8 | Grafana Top Talkers shows data | Navigate to Flow — Top Talkers in Grafana | Table populated with src_ip/bytes | [ ] |
| 3.9 | Repoint remaining hosts | Update softflowd on all non-canary hosts | All restarted, pointing to FLOW_COLLECTOR_IP | [ ] |
| 3.10 | All hosts exporting | Check exporter bucket agg in OpenSearch | One bucket per active Proxmox host | [ ] |
| 3.11 | Waves 1–2 still healthy | Re-run checks 1.3–1.7, 2.1–2.4 | All still passing | [ ] |

**Gate:** All items pass → proceed to Wave 3b (K8s Correlation) or Wave 4 (K8s Visibility).

**Rollback trigger:** GoFlow2 file empty after 5 minutes. Vector indexing errors. Corrupted flow data. doc count not growing.

**Rollback:** `ssh prox5; sudo cp /etc/default/softflowd.bak /etc/default/softflowd; sudo systemctl restart softflowd`. Repeat on any other repointed host. Legacy collector at `172.18.1.207:2055` resumes immediately.

---

## Wave 3b — Kubernetes Correlation

**Prerequisite:** Gate 3 (Flows Ingesting) must pass. Cilium is NOT required for this wave.

**Note:** Wave 3b is the K8s Correlation wave defined in [wave-3-plan.md](wave-3-plan.md).
It adds Kubernetes entity enrichment to flow documents using a CronJob-managed
lookup table in Vector. It does not require Cilium or Hubble.

**Phase 1 gate** (design complete — before any manifests are applied):

| # | Check | How | Expected | Pass |
|---|---|---|---|---|
| 3b.0 | Pod CIDR confirmed from cluster config | `kubectl get node <node> -o jsonpath='{.spec.podCIDR}'` and/or `ssh <control-plane> grep service-cluster-ip-range /etc/kubernetes/manifests/kube-apiserver.yaml` | CIDR documented from authoritative source | [ ] |
| 3b.1 | Service CIDR confirmed from cluster config | `kubectl describe cm kubeadm-config -n kube-system \| grep serviceSubnet` or kube-apiserver manifest | CIDR documented from authoritative source | [ ] |
| 3b.2 | Node IPs enumerated | `kubectl get nodes -o wide` — record INTERNAL-IP column | All node IPs listed | [ ] |
| 3b.3 | Index template draft reviewed | Review proposed field additions including `src_k8s_node`, `dst_k8s_node`, `src_k8s_service`, `dst_k8s_service` | No field type conflicts with existing fields | [ ] |
| 3b.4 | Lookup table schema approved | Review wave-3-plan.md Phase 1 output including `_timestamp` column and `service` column in k8s-services.csv | Schema agreed including all columns | [ ] |
| 3b.23 | Restart model accepted | Confirm lifecycle contract in wave-3-plan.md: Vector 0.45.0 restart-required, atomic-rename-then-single-restart pattern, no live-reload | Restart model documented and accepted before Phase 2 manifests begin | [ ] |

**Phase 2 gate** (enrichment active — after manifests applied):

Scratch-index validation must complete before production template apply (item 3b.7).

| # | Check | Command | Expected | Pass |
|---|---|---|---|---|
| 3b.5 | CronJob deployed | `kubectl get cronjob -n network-observability` | k8s-ip-exporter present | [ ] |
| 3b.6 | CronJob has run successfully | `kubectl get jobs -n network-observability` | At least 1 successful job | [ ] |
| 3b.7 | **Scratch-index validation** | Create temp index with new template, index synthetic doc with all K8s fields, verify no mapping exceptions, delete scratch index | 0 mapping errors on all new fields before production apply | [ ] |
| 3b.8 | Production index template applied | `curl localhost:9200/_index_template/flows` | Template includes `src_k8s_node`, `dst_k8s_node`, `src_k8s_service`, `dst_k8s_service` fields | [ ] |
| 3b.9 | All lookup CSVs non-empty | `kubectl exec deploy/flow-collector -c vector -- wc -l /etc/vector/k8s-pods.csv /etc/vector/k8s-services.csv /etc/vector/k8s-nodes.csv` | All files > 1 line | [ ] |
| 3b.10 | Vector config validates | Run validate pod against production image | `Validated`, 0 errors | [ ] |
| 3b.11 | Enriched document exists | `curl localhost:9200/flows-*/_search?q=src_k8s_namespace:*&size=1&pretty` | Document with `src_k8s_namespace` field present | [ ] |
| 3b.12 | Node field present on pod-type doc | Check document where `src_k8s_type: pod` | `src_k8s_node` field populated | [ ] |
| 3b.13 | internal-unknown classification works | Find a CIDR-internal IP with no pod/service row | `src_k8s_type: internal-unknown` (not `external`) | [ ] |
| 3b.14 | Wave 2 still healthy | Re-run checks 2.1–2.4, confirm doc count growing | All passing | [ ] |
| 3b.24 | CronJob restart mechanism validated | Confirm CronJob performs atomic rename then single pod restart; check CronJob logs show restart only after all CSV files valid | Restart occurs once per cycle, only on all-valid outcome | [ ] |
| 3b.25 | hostNetwork pods excluded from pod CSV | Inspect k8s-pods.csv — confirm no rows with a node IP (172.18.1.61–.64) as the IP column | No node IPs appear in pod CSV | [ ] |

**Phase 3 gate** (dashboards validated):

| # | Check | Action | Expected | Pass |
|---|---|---|---|---|
| 3b.15 | K8s Flow Context dashboard loads | Navigate to dashboard in Grafana | Loads in < 5s | [ ] |
| 3b.16 | Namespace aggregation populated | Check namespace breakdown panel | Shows namespaces with bytes | [ ] |
| 3b.17 | internal-unknown bucket visible | Check `src_k8s_type` breakdown panel | `internal-unknown` appears as distinct bucket (not merged with external) | [ ] |
| 3b.18 | External IPs handled gracefully | Confirm external IP rows have empty K8s fields | No errors or broken panels | [ ] |
| 3b.19 | Entity Investigation shows K8s fields | Open Entity Investigation for a pod IP | `src_k8s_*` fields including node name visible | [ ] |

**Phase 4 gate** (soak complete):

| # | Check | Command | Expected | Pass |
|---|---|---|---|---|
| 3b.20 | CronJob ran 3+ successful cycles | `kubectl get jobs -n network-observability` | 3+ Completed | [ ] |
| 3b.21 | No Vector enrichment errors | `kubectl logs deploy/flow-collector -c vector \| grep -i 'error\|warn'` | No new errors vs baseline | [ ] |
| 3b.22 | ArgoCD sync clean | `argocd app get network-observability` | Synced, Healthy | [ ] |

**Gate:** All Phase 1–4 items (3b.0–3b.25) pass → Gate 3b met. Proceed to Wave 4 or Wave 5.

**Rollback trigger:** Vector enrichment errors appear after table changes. Existing
flow fields become null or change type. Document count drops significantly.

**Rollback:** Remove enrichment table config from `vector-config`, remove CronJob,
redeploy. Existing Wave 2 flow data in OpenSearch is unaffected — the new fields
will simply be absent on older documents.

---

## Wave 4 — K8s Visibility *(DEFERRED)*

> **Wave 4 is deferred.** Cilium migration has not been scheduled. Skip to Wave 5.
> The full phased checklist below is preserved for when Wave 4 is unblocked.
> See [wave-4-plan.md](wave-4-plan.md) and backlog.md for details.

**Prerequisite:** Cilium CNI with Hubble must be installed and healthy before
Phases 3–5. If not yet migrated, complete Phases 1–2 first. Wave 4 may be deferred
entirely — skip to Wave 5 if the maintenance window is not available.

Wave 4 is phased. Phases 1–2 are discovery and migration (no manifests from this
repo). Phases 3–5 are implementation and soak.

### Phase 1 — Discovery

**No cluster changes. Queries only. Results must be documented before Phase 2 begins.**

| # | Check | Command | Expected | Pass |
|---|---|---|---|---|
| 4.1.1 | Identify current CNI | `kubectl get pods -n kube-system -o wide \| grep -E 'calico\|cilium\|flannel\|canal'` and `kubectl get ds -n kube-system` | CNI identified and documented | [ ] |
| 4.1.2 | Confirm pod CIDR | `kubectl get node -o jsonpath='{.items[0].spec.podCIDR}'` | `10.244.0.0/16` (must match Wave 3b VRL) | [ ] |
| 4.1.3 | Confirm service CIDR | `kubectl describe cm kubeadm-config -n kube-system \| grep serviceSubnet` | `10.96.0.0/12` (must match Wave 3b VRL) | [ ] |
| 4.1.4 | Audit NetworkPolicies | `kubectl get networkpolicies --all-namespaces` | Count documented; any Calico-specific types noted | [ ] |
| 4.1.5 | Check Calico-specific resources | `kubectl get globalnetworkpolicies 2>/dev/null; kubectl get networksets 2>/dev/null` | None (or documented if present) | [ ] |
| 4.1.6 | Inventory hostNetwork pods | `kubectl get pods --all-namespaces -o json \| jq '.items[] \| select(.spec.hostNetwork==true) \| .metadata.namespace + "/" + .metadata.name'` | Count documented | [ ] |
| 4.1.7 | Confirm cluster health | `kubectl get nodes` and `kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded` | All nodes Ready; no stuck non-system pods | [ ] |
| 4.1.8 | Record Wave 3b baseline | `kubectl get jobs -n network-observability --sort-by=.metadata.creationTimestamp \| tail -3` | At least 1 recent successful CronJob run documented | [ ] |
| 4.1.9 | Document maintenance window | (Ops decision) | Agreed window recorded | [ ] |
| 4.1.10 | Record migration go/no-go | (Ops decision) | Proceed to Phase 2 or defer; decision documented | [ ] |

**Gate 4a:** All items pass and CIDRs match Wave 3b assumptions → proceed to Phase 2.
**If deferred:** Document decision and proceed to Wave 5.

---

### Phase 2 — Cilium Migration

**This phase is executed in `k8s-lab.git`, not this repo. These checks confirm
readiness before Phase 3 manifests are created.**

**Pre-migration backup (run before any migration work):**

```bash
kubectl get nodes -o wide > /tmp/nodes-before-migration.txt
kubectl get pods --all-namespaces -o wide > /tmp/pods-before-migration.txt
kubectl get networkpolicies --all-namespaces -o yaml > /tmp/netpol-backup.yaml
```

| # | Check | Command | Expected | Pass |
|---|---|---|---|---|
| 4.2.1 | Backup saved | See commands above | Three backup files written | [ ] |
| 4.2.2 | Cilium DaemonSet healthy | `kubectl -n kube-system get ds cilium` | DESIRED = READY (all nodes) | [ ] |
| 4.2.3 | Cilium operator healthy | `kubectl -n kube-system get deploy cilium-operator` | 1/1 Ready | [ ] |
| 4.2.4 | Hubble Relay healthy | `kubectl -n kube-system get deploy hubble-relay` | 1/1 Ready | [ ] |
| 4.2.5 | Hubble enabled in config | `kubectl -n kube-system get cm cilium-config -o jsonpath='{.data.enable-hubble}'` | `"true"` | [ ] |
| 4.2.6 | All cluster pods running | `kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded` | Only Completed (Job) pods | [ ] |
| 4.2.7 | Cross-namespace Relay reachable | `kubectl run relay-test --rm -i --image=busybox -n network-observability -- nc -zv hubble-relay.kube-system.svc.cluster.local 80` | Connection succeeded | [ ] |
| 4.2.8 | Pod CIDR unchanged | `kubectl get node -o jsonpath='{.items[0].spec.podCIDR}'` | `10.244.0.0/16` (same as pre-migration) | [ ] |
| 4.2.9 | Service CIDR unchanged | `kubectl describe cm kubeadm-config -n kube-system \| grep serviceSubnet` | `10.96.0.0/12` (same as pre-migration) | [ ] |
| 4.2.10 | Wave 3b CronJob still healthy | `kubectl get jobs -n network-observability --sort-by=.metadata.creationTimestamp \| tail -3` | At least 1 successful job post-migration | [ ] |
| 4.2.11 | Enriched flows present post-migration | `curl localhost:9200/flows-*/_search?q=src_k8s_type:pod&size=1&pretty \| grep src_k8s` | src_k8s fields present in recent flow documents | [ ] |

**Gate 4b:** All items pass → proceed to Phase 3 (Hubble UI deployment).

---

### Phase 3 — Hubble UI Deployment

**Enable:** Uncomment `../../base/k8s-visibility` in `platform/overlays/lab/kustomization.yaml`, commit, push. Wait for ArgoCD sync.

| # | Check | Command | Expected | Pass |
|---|---|---|---|---|
| 4.3.1 | Plane enabled in overlay | Verify k8s-visibility line is uncommented in lab kustomization.yaml | Uncommented | [ ] |
| 4.3.2 | ArgoCD sync status | `argocd app get network-observability` | Synced, Healthy | [ ] |
| 4.3.3 | Hubble UI Running | `kubectl get pods -n network-observability -l app.kubernetes.io/name=hubble-ui` | 1/1 Running | [ ] |
| 4.3.4 | Port-forward Hubble UI | `kubectl port-forward svc/hubble-ui 12000:80 -n network-observability` | Accessible, no error | [ ] |
| 4.3.5 | Hubble UI loads | Open `http://localhost:12000` | UI renders, no "Relay unavailable" error | [ ] |
| 4.3.6 | Pod flows visible | Select any active namespace in Hubble UI | Pod-to-pod flows shown | [ ] |
| 4.3.7 | DNS flows visible | Select `kube-system` or any DNS-active namespace | DNS query flows shown | [ ] |
| 4.3.8 | Two namespaces visible | Check namespace dropdown | At least two namespaces with flows | [ ] |
| 4.3.9 | Verdict visible | Examine any flow | Verdict field shows "forwarded" or "dropped" | [ ] |
| 4.3.10 | Wave 3b regression | `curl localhost:9200/flows-*/_search?q=src_k8s_type:pod&size=1&pretty` | Enriched K8s fields still present | [ ] |

---

### Phase 4 — Dashboard and UX Integration

| # | Check | Action | Expected | Pass |
|---|---|---|---|---|
| 4.4.1 | Hubble Prometheus metrics present | `kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring` then query `hubble_` metrics | hubble_drop_total, hubble_flows_processed_total, hubble_dns_* present | [ ] |
| 4.4.2 | Hubble dashboard provisioned | Navigate to Grafana → Dashboards | Hubble metrics dashboard visible | [ ] |
| 4.4.3 | Drop rate panel populated | Check drop rate panel | Renders without error; shows 0 or actual drops | [ ] |
| 4.4.4 | Namespace flow volume populated | Check namespace flow breakdown panel | Shows at least two namespaces with flow counts | [ ] |
| 4.4.5 | Platform Health shows Hubble Relay | Navigate to Platform Health dashboard | Hubble Relay shown as monitored target | [ ] |
| 4.4.6 | Operations notes updated | Read ops-notes entry for Wave 4 | Port-forward pattern documented | [ ] |

---

### Phase 5 — Soak

| # | Check | Command | Expected | Pass |
|---|---|---|---|---|
| 4.5.1 | 7-day soak clean | `kubectl get pods -n network-observability` (check restarts) | Hubble UI restart count stable | [ ] |
| 4.5.2 | No Relay reconnect loops | `kubectl logs -n network-observability deploy/hubble-ui \| grep -i 'error\|warn\|reconnect'` | No reconnect loops or persistent errors | [ ] |
| 4.5.3 | Wave 3b still healthy | `kubectl get jobs -n network-observability --sort-by=.metadata.creationTimestamp \| tail -5` | All recent CronJob runs successful | [ ] |
| 4.5.4 | OpenSearch doc count growing | `curl localhost:9200/flows-*/_count` | Count higher than at Phase 3 baseline | [ ] |
| 4.5.5 | ArgoCD sync clean | `argocd app get network-observability` | Synced, Healthy | [ ] |

**Gate 4c:** All Phase 3–5 items pass, 7-day soak clean → Gate 4 met. Proceed to Wave 5.

**Rollback trigger:** Hubble UI shows persistent "Relay unavailable" after investigation.
Wave 3b regression (enriched fields absent, CronJob failing).

**Hubble UI rollback:** Comment out `../../base/k8s-visibility` in overlay, commit, push.
ArgoCD auto-prune removes Hubble UI. Wave 3b unaffected.

**Cilium rollback:** Managed in `k8s-lab.git`. See cni-migration-and-rollout.md rollback
section. Wave 3b resumes automatically after pod restart cycle.

---

## Wave 5 — Unified UX Go-Live *(ACTIVE)*

**Baseline:** Waves 1, 2, 3, 3b complete. Wave 4 deferred.

Wave 5 builds the unified operator UX across the three stable data planes (infra
telemetry, flow analytics, K8s enrichment) using Grafana as the primary surface.
No Cilium required. Wave 4 Hubble integration activates as an optional add-on
if Wave 4 is completed later.

**Enable (Phase 3):** Uncomment `../../base/correlation-ux` in
`platform/overlays/lab/kustomization.yaml`, commit, push. Wait for ArgoCD sync.

### Phase 1 — Baseline Audit

**No new deployments. Verify all existing dashboards are healthy before building on top.**

| # | Check | Command / Action | Expected | Pass |
|---|---|---|---|---|
| 5.1.1 | Prometheus datasource healthy | Grafana → Data Sources → Prometheus → Test | Connection successful | [x] |
| 5.1.2 | OpenSearch datasource healthy | Grafana → Data Sources → OpenSearch Flows → Test | Connection successful | [x] |
| 5.1.3 | Infra dashboards load with data | Open Switch Interface Util, UniFi AP, Proxmox dashboards | All three load with live data, no broken panels | [x] |
| 5.1.4 | Flow dashboards load with data | Open Top Talkers, Destination Analysis, Traffic Mix | All three load with data, OpenSearch queries returning rows | [x] |
| 5.1.5 | Wave 3b CronJob healthy | `kubectl get jobs -n network-observability --sort-by=.metadata.creationTimestamp \| tail -3` | Recent successful runs | [x] |
| 5.1.6 | K8s fields present in flows | `curl localhost:9200/flows-*/_search?q=src_k8s_type:pod&size=1&pretty \| grep src_k8s` | src_k8s fields populated | [x] |
| 5.1.7 | Document baseline flow count | `curl localhost:9200/flows-*/_count` | Record count for regression comparison | [x] |

**Gate 5a pre-work:** All items pass → proceed to Phase 2.

---

### Phase 2 — K8s Flow Context Dashboard

**Scope:** Build and deploy the K8s Flow Context dashboard (Wave 3b backlog item,
now active). Add a ConfigMap to `platform/base/flow-analytics/` and include it in
kustomization.yaml. ArgoCD deploys it automatically.

| # | Check | Action | Expected | Pass |
|---|---|---|---|---|
| 5.2.1 | Dashboard ConfigMap created | Review new Grafana dashboard ConfigMap in flow-analytics plane | ConfigMap has correct labels for Grafana sidecar provisioning | [x] |
| 5.2.2 | ArgoCD sync picks up new dashboard | `argocd app get network-observability` | Synced, Healthy | [x] |
| 5.2.3 | Dashboard appears in Grafana | Grafana → Dashboards → Browse | "K8s Flow Context" dashboard listed | [x] |
| 5.2.4 | Namespace breakdown panel | Open dashboard, check namespace panel | Shows at least two K8s namespaces with flow bytes | [x] |
| 5.2.5 | Pod-to-pod flow panel | Check top pod flows panel | Shows src\_k8s\_pod/dst\_k8s\_pod pairs with bytes | [x] |
| 5.2.6 | internal-unknown rate panel | Check classification breakdown | `internal-unknown` visible as distinct bucket, not merged into external | [x] |
| 5.2.7 | Service-type flow panel | Check service flow panel | service-type flows visible with namespace and service name | [x] |
| 5.2.8 | External flows panel | Check external panel | External IPs shown with no broken panels (K8s fields empty — expected) | [x] |
| 5.2.9 | Wave 3b regression | Re-run 5.1.5 and 5.1.6 | CronJob still healthy, enriched fields still present | [x] |

**Gate 5a:** Phases 1–2 complete → K8s Flow Context dashboard operational.

> **Gate 5a: ACCEPTED WITH MINOR FOLLOW-UP — 2026-04-26.**
> Seven-dashboard baseline accepted. See go-no-go-gates.md for verdict and conditions.

---

### Phase 3 — Homepage and Navigation

**Enable:** Uncomment `../../base/correlation-ux` in
`platform/overlays/lab/kustomization.yaml`, commit, push. ArgoCD syncs the plane.

| # | Check | Command / Action | Expected | Pass |
|---|---|---|---|---|
| 5.3.1 | Plane enabled in overlay | Verify correlation-ux line is uncommented in lab kustomization.yaml | Uncommented and committed | [ ] |
| 5.3.2 | ArgoCD sync healthy | `argocd app get network-observability` | Synced, Healthy | [ ] |
| 5.3.3 | Homepage loads | Navigate to `/d/correlation-home` in Grafana | Loads in < 5s | [ ] |
| 5.3.4 | Card 1: Top Talkers populated | Check Top Talkers table | At least one row with src\_ip and bytes (not empty) | [ ] |
| 5.3.5 | Card 2: Destinations populated | Select a src\_ip from Top Talkers, check Destinations card | Destination rows appear with dst\_ip, port, country | [ ] |
| 5.3.6 | Card 3: WiFi clients populated | Check WiFi client bandwidth card | At least one client visible with RX/TX values | [ ] |
| 5.3.7 | Freshness bar — healthy state | All exporters running — check freshness bar | All indicators green | [ ] |
| 5.3.8 | Freshness bar — degraded state | Stop one exporter, wait 2 scrape cycles, check bar | Degraded indicator appears for affected source | [ ] |
| 5.3.9 | Restart exporter, freshness recovers | Restart the stopped exporter | Green indicator returns within 2 scrape cycles | [ ] |
| 5.3.10 | Specialist Views dropdown | Click Specialist Views in homepage nav | K8s Flow Context and Entity Investigation appear in list | [ ] |
| 5.3.11 | Investigation Playbooks dropdown | Click Investigation Playbooks in homepage nav | Playbooks dashboard or list appears | [ ] |

---

### Phase 4 — Entity Investigation

| # | Check | Action | Expected | Pass |
|---|---|---|---|---|
| 5.4.1 | Click IP in Top Talkers | Click a src\_ip row in Top Talkers card | Opens Entity Investigation with IP variable pre-filled | [ ] |
| 5.4.2 | Enter pod IP manually | Enter a known pod IP (10.244.x.x) in IP variable | Entity Investigation loads | [ ] |
| 5.4.3 | K8s context for pod IP | Check K8s fields section | Namespace, workload, pod, node populated from Wave 3b enrichment | [ ] |
| 5.4.4 | Outbound flow table | Check outbound flows section | Shows dst\_ip, dst\_port, protocol, bytes for the entity | [ ] |
| 5.4.5 | Inbound flow table | Check inbound flows section | Shows src\_ip rows for traffic hitting the entity | [ ] |
| 5.4.6 | Flow volume time series | Check volume chart | Shows bytes/packets over time for the entity | [ ] |
| 5.4.7 | Link to Destination Analysis | Click "View in Flow — Destinations" link | Opens Flow — Destination Analysis with src\_ip pre-filled | [ ] |
| 5.4.8 | Hubble link present but inactive | Check Hubble link (if present in dashboard) | Link is visible; correctly indicates Hubble not available or port-forward needed | [ ] |
| 5.4.9 | External IP handling | Enter an external IP (e.g., 8.8.8.8) | Entity Investigation loads; K8s fields empty (expected); flows present if seen | [ ] |

---

### Phase 5 — Platform Health Dashboard

| # | Check | Action | Expected | Pass |
|---|---|---|---|---|
| 5.5.1 | Platform Health loads | Navigate to `/d/correlation-platform-health` | Loads in < 5s | [ ] |
| 5.5.2 | All scrape targets shown | Check targets table | SNMP, UnPoller, Proxmox all listed with UP status | [ ] |
| 5.5.3 | Flow collector shown | Check flow collector target | flow-collector (Vector) shown as UP | [ ] |
| 5.5.4 | OpenSearch shown | Check OpenSearch target | OpenSearch listed as reachable | [ ] |
| 5.5.5 | Wave 3b CronJob shown | Check CronJob last-success indicator | Last successful run within expected cadence | [ ] |
| 5.5.6 | Degraded state test | Stop an exporter | Platform Health shows that target as DOWN | [ ] |
| 5.5.7 | Recovery | Restart the exporter | Status returns to UP within 2 scrape cycles | [ ] |

**Gate 5b:** Phases 3–5 complete → homepage, navigation, entity investigation, and
health dashboard all operational.

---

### Phase 6 — Investigation Playbooks

| # | Check | Action | Expected | Pass |
|---|---|---|---|---|
| 5.6.1 | Playbooks dashboard loads | Navigate to Investigation Playbooks dashboard | Loads in < 5s | [ ] |
| 5.6.2 | All 6 playbooks listed | Check playbook list | "Investigate slow internet", "Identify top talker", "Trace a destination", "K8s workload traffic", "Unusual port/protocol", "Platform health check" — all 6 present | [ ] |
| 5.6.3 | "Investigate slow internet" executable | Follow playbook steps | All steps navigable; links resolve to correct dashboards | [ ] |
| 5.6.4 | "K8s workload traffic" executable | Follow playbook steps | Steps include K8s Flow Context dashboard and Entity Investigation with pod IP | [ ] |
| 5.6.5 | Operator end-to-end walkthrough | Operator executes one playbook start-to-finish | Operator confirms playbook is complete and actionable | [ ] |

---

### Phase 7 — Soak and Closeout

| # | Check | Command / Action | Expected | Pass |
|---|---|---|---|---|
| 5.7.1 | 7-day soak: homepage | Daily spot-check: `/d/correlation-home` loads | No load failures across 7 days | [ ] |
| 5.7.2 | 7-day soak: data sources | Daily spot-check: both datasources healthy | No persistent datasource errors | [ ] |
| 5.7.3 | 7-day soak: Wave 3b regression | `kubectl get jobs -n network-observability --sort-by=.metadata.creationTimestamp \| tail -5` | All recent CronJob runs successful | [ ] |
| 5.7.4 | 7-day soak: doc count | `curl localhost:9200/flows-*/_count` | Count higher than Phase 1 baseline | [ ] |
| 5.7.5 | ArgoCD clean at closeout | `argocd app get network-observability` | Synced, Healthy | [ ] |
| 5.7.6 | All prior wave checks pass | Spot-check items from Waves 1, 2, 3, 3b | No regressions | [ ] |
| 5.7.7 | wave-5-closeout.md written | Review doc | Executive summary, evidence, caveats, acceptance statement present | [ ] |

**Gate 5c:** All soak items pass, closeout doc written → Gate 5 met. Platform go-live.

**Rollback trigger:** Homepage consistently fails to load. Both datasources erroring.
Wave 3b enrichment fields dropping out during soak.

**Rollback:** Comment out `../../base/correlation-ux` in overlay, commit, push.
ArgoCD prunes all UX ConfigMaps. Grafana reverts to plain Prometheus + OpenSearch
datasources. All data planes (Wave 1–3b) unaffected.
