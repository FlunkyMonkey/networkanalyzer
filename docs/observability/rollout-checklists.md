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

## Wave 4 — K8s Visibility

**Prerequisite:** Cilium CNI with Hubble must be installed before this wave. If not yet migrated, skip Wave 4 and proceed to Wave 5.

**Enable:** Uncomment `../../base/k8s-visibility` in `platform/overlays/lab/kustomization.yaml`, commit, push. Wait for ArgoCD sync.

| # | Check | Command | Expected | Pass |
|---|---|---|---|---|
| 4.0a | Cilium running | `kubectl -n kube-system get ds cilium` | DESIRED = READY | [ ] |
| 4.0b | Hubble Relay running | `kubectl -n kube-system get deploy hubble-relay` | 1/1 Ready | [ ] |
| 4.1 | Hubble UI Running | `kubectl get pods -n network-observability -l app.kubernetes.io/name=hubble-ui` | Running | [ ] |
| 4.2 | Port-forward Hubble UI | `kubectl port-forward svc/hubble-ui 12000:80 -n network-observability` | Accessible | [ ] |
| 4.3 | Service map visible | Open <http://localhost:12000> | Shows service map | [ ] |
| 4.4 | Pod flows visible | Filter by any namespace | Pod-to-pod flows shown | [ ] |
| 4.5 | DNS visible | Check for DNS flows in Hubble | DNS queries shown (if enabled) | [ ] |
| 4.6 | Waves 1–3 still healthy | Re-run checks 1.3–1.7, 2.1–2.4, 3.8 | All passing | [ ] |

**Gate:** All items pass → proceed to Wave 5.

**Rollback trigger:** Hubble UI shows "Relay unavailable" persistently.

---

## Wave 5 — Unified UX Go-Live

**Enable:** Uncomment `../../base/correlation-ux` in `platform/overlays/lab/kustomization.yaml`, commit, push. Wait for ArgoCD sync.

| # | Check | Command / Action | Expected | Pass |
|---|---|---|---|---|
| 5.0 | Plane enabled in overlay | Verify correlation-ux line is uncommented in lab kustomization.yaml | Uncommented | [ ] |
| 5.1 | Homepage loads | Navigate to `/d/correlation-home` | Loads in < 5s | [ ] |
| 5.2 | Card 1: Top Talkers | Check top talkers table | Shows IPs with bytes | [ ] |
| 5.3 | Card 2: Destinations | Enter a source IP, check destination table | Shows dst IPs, ports, countries | [ ] |
| 5.4 | Card 3: WiFi clients | Check WiFi bandwidth chart | Shows client RX/TX | [ ] |
| 5.5 | Freshness bar | Check source health / scrape % / flow ingest | All green/healthy | [ ] |
| 5.6 | Drill-down: IP → Entity | Click an IP in Top Talkers | Opens Entity Investigation | [ ] |
| 5.7 | Drill-down: IP → Flow Destinations | Click "View flow destinations" link on Top Talkers row | Opens Flow — Destination Analysis with src_ip pre-filled | [ ] |
| 5.8 | Playbooks accessible | Click "Investigation Playbooks" dropdown | All 6 playbooks visible | [ ] |
| 5.9 | Health dashboard | Navigate to Platform Health | All targets shown UP | [ ] |
| 5.10 | End-to-end playbook | Execute "Investigate slow internet" | Can follow all steps | [ ] |
| 5.11 | Waves 1–4 still healthy | Spot-check key items from each wave | All passing | [ ] |

**Gate:** All items pass → platform is go-live.

**Rollback trigger:** Homepage fails to load. Critical drill-down paths broken. Data sources not populating.
