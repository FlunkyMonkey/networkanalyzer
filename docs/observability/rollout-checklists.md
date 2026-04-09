# Rollout Checklists

Operator-executable checklists for each rollout wave. Print, execute line by line, and sign off.

## Wave 0 — Preflight

**Already completed:** Namespace, grafana-admin-credentials, unpoller-credentials, proxmox-credentials.

| # | Check | Command | Expected | Pass |
|---|---|---|---|---|
| 0.1 | Kustomize renders | `kustomize build --enable-helm platform/overlays/lab \| grep '^kind:' \| wc -l` | ~149 objects, no errors | [ ] |
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

| # | Check | Command | Expected | Pass |
|---|---|---|---|---|
| 1.1 | Apply ArgoCD app | `kubectl apply -f platform/apps/network-observability.yaml` | Application created | [ ] |
| 1.2 | ArgoCD sync status | `argocd app get network-observability` | Synced, Healthy | [ ] |
| 1.3 | All pods Running | `kubectl get pods -n network-observability` | All Running/Ready | [ ] |
| 1.4 | Prometheus up | `kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 -n network-observability` then check `/targets` | All targets UP | [ ] |
| 1.5 | SNMP exporter scraping | Check Prometheus targets for snmp-exporter | UP, last scrape < 2m | [ ] |
| 1.6 | UnPoller scraping | Check Prometheus targets for unpoller | UP, last scrape < 2m | [ ] |
| 1.7 | Proxmox exporter scraping | Check Prometheus targets for proxmox-exporter | UP, last scrape < 2m | [ ] |
| 1.8 | Grafana accessible | `kubectl port-forward svc/prometheus-grafana 3000:80 -n network-observability` | Login page loads | [ ] |
| 1.9 | Interface dashboard works | Navigate to Switch Interface Utilization | Shows live data | [ ] |
| 1.10 | UniFi dashboard works | Navigate to UniFi AP & WLAN Clients | Shows live data | [ ] |
| 1.11 | Proxmox dashboard works | Navigate to Proxmox Node & VM Network | Shows live data | [ ] |
| 1.12 | PVCs bound | `kubectl get pvc -n network-observability` | All Bound | [ ] |

**Gate:** All items pass → proceed to Wave 2.

**Rollback trigger:** Prometheus, Grafana, or any exporter fails to become healthy after 10 minutes.

**Rollback:** `kubectl delete -f platform/apps/network-observability.yaml && kubectl delete pvc --all -n network-observability`

---

## Wave 2 — Flow Plane Infrastructure

| # | Check | Command | Expected | Pass |
|---|---|---|---|---|
| 2.1 | OpenSearch Running | `kubectl get pods -n network-observability -l app.kubernetes.io/name=opensearch` | 1/1 Running | [ ] |
| 2.2 | OpenSearch health | Port-forward 9200, `curl localhost:9200/_cluster/health` | `"status":"green"` | [ ] |
| 2.3 | Dashboards Running | `kubectl get pods -n network-observability -l app.kubernetes.io/name=opensearch-dashboards` | Running | [ ] |
| 2.4 | Flow collector Running | `kubectl get pods -n network-observability -l app.kubernetes.io/name=flow-collector` | 2/2 Running | [ ] |
| 2.5 | Collector external IP | `kubectl get svc flow-collector -n network-observability` | EXTERNAL-IP assigned | [ ] |
| 2.6 | Index template loaded | `curl localhost:9200/_index_template/flows` | Template exists | [ ] |
| 2.7 | ISM policy loaded | `curl localhost:9200/_plugins/_ism/policies/flow-retention-30d` | Policy exists | [ ] |
| 2.8 | Dashboards objects imported | Access Dashboards at :5601, check index pattern `flows-*` | Exists | [ ] |
| 2.9 | Wave 1 still healthy | Re-run checks 1.3–1.7 | All still passing | [ ] |

**Gate:** All items pass → proceed to Wave 3.

**Rollback trigger:** OpenSearch PVC fails to bind. Flow collector cannot start. MetalLB fails to assign IP.

---

## Wave 3 — Softflowd Cutover

| # | Check | Command | Expected | Pass |
|---|---|---|---|---|
| 3.1 | Record current softflowd targets | `ssh each-host cat /etc/default/softflowd` | Targets documented | [ ] |
| 3.2 | Canary: repoint 1 host | Update softflowd config on 1 host, restart | Service restarted | [ ] |
| 3.3 | GoFlow2 receiving | `kubectl exec deploy/flow-collector -c goflow2 -- tail -3 /flows/flows.jsonl` | JSON records appearing | [ ] |
| 3.4 | OpenSearch indexing | `curl localhost:9200/flows-*/_count` | Count > 0 and growing | [ ] |
| 3.5 | Flow fields correct | `curl 'localhost:9200/flows-*/_search?size=1&pretty'` | src_ip, dst_ip, bytes present | [ ] |
| 3.6 | GeoIP enrichment (if configured) | Check for dst_country field in flow doc | Present if MaxMind key was set | [ ] |
| 3.7 | Repoint remaining hosts | Update softflowd config on all remaining hosts | All restarted | [ ] |
| 3.8 | All hosts exporting | Check flow count growth rate | Steady increase | [ ] |
| 3.9 | OpenSearch Dashboards show data | Query `flows-*` in Discover | Results returned | [ ] |
| 3.10 | Waves 1–2 still healthy | Re-run checks 1.3–1.7, 2.1–2.4 | All still passing | [ ] |

**Gate:** All items pass → proceed to Wave 4.

**Rollback trigger:** Flows not arriving after 5 minutes. Corrupted data. OpenSearch indexing errors.

**Rollback:** Repoint softflowd back to original target on all hosts. Restart softflowd.

---

## Wave 4 — K8s Visibility

**Prerequisite:** Cilium CNI with Hubble must be installed before this wave. If not yet migrated, skip Wave 4 and proceed to Wave 5.

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

| # | Check | Command / Action | Expected | Pass |
|---|---|---|---|---|
| 5.1 | Homepage loads | Navigate to `/d/correlation-home` | Loads in < 5s | [ ] |
| 5.2 | Card 1: Top Talkers | Check top talkers table | Shows IPs with bytes | [ ] |
| 5.3 | Card 2: Destinations | Enter a source IP, check destination table | Shows dst IPs, ports, countries | [ ] |
| 5.4 | Card 3: WiFi clients | Check WiFi bandwidth chart | Shows client RX/TX | [ ] |
| 5.5 | Freshness bar | Check source health / scrape % / flow ingest | All green/healthy | [ ] |
| 5.6 | Drill-down: IP → Entity | Click an IP in Top Talkers | Opens Entity Investigation | [ ] |
| 5.7 | Drill-down: IP → OpenSearch | Click "View flows in OpenSearch" link | Opens OSD with filter | [ ] |
| 5.8 | Playbooks accessible | Click "Investigation Playbooks" dropdown | All 6 playbooks visible | [ ] |
| 5.9 | Health dashboard | Navigate to Platform Health | All targets shown UP | [ ] |
| 5.10 | End-to-end playbook | Execute "Investigate slow internet" | Can follow all steps | [ ] |
| 5.11 | Waves 1–4 still healthy | Spot-check key items from each wave | All passing | [ ] |

**Gate:** All items pass → platform is go-live.

**Rollback trigger:** Homepage fails to load. Critical drill-down paths broken. Data sources not populating.
