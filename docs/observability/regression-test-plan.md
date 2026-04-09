# Regression Test Plan

Defines what must still work after each rollout wave. Run these checks after completing each wave to ensure no regressions were introduced.

## After Wave 1 (Telemetry Plane)

| # | Check | How to Verify |
|---|---|---|
| R1.1 | Cluster health | `kubectl get nodes` — all Ready |
| R1.2 | Existing workloads unaffected | `kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded` — only expected non-Running pods |
| R1.3 | ArgoCD healthy | `argocd app list` — no degraded apps |
| R1.4 | DNS resolution working | `kubectl run dns-test --rm -i --image=busybox -- nslookup kubernetes.default` |
| R1.5 | Pod networking intact | `kubectl run net-test --rm -i --image=busybox -- wget -qO- http://kubernetes.default/healthz` |

## After Wave 2 (Flow Plane Infrastructure)

All Wave 1 regressions plus:

| # | Check | How to Verify |
|---|---|---|
| R2.1 | Prometheus still scraping | Prometheus Targets page — all targets UP |
| R2.2 | Grafana still accessible | Port-forward, login works |
| R2.3 | Infra dashboards still show data | Check interface utilization dashboard |
| R2.4 | PVC usage not saturated | `kubectl get pvc -n network-observability` — capacity not exceeded |
| R2.5 | Node resource pressure | `kubectl top nodes` — no CPU/memory pressure |

## After Wave 3 (Softflowd Cutover)

All Wave 1 + Wave 2 regressions plus:

| # | Check | How to Verify |
|---|---|---|
| R3.1 | Softflowd still running on all hosts | `ssh each-host systemctl status softflowd` |
| R3.2 | No flow data loss | Flow count in OpenSearch is growing steadily |
| R3.3 | Infra metrics unaffected | SNMP, UnPoller, Proxmox targets still UP in Prometheus |
| R3.4 | OpenSearch health | `curl localhost:9200/_cluster/health` — green |
| R3.5 | Grafana dashboards still load | Check all 3 infra dashboards + homepage |

## After Wave 4 (K8s Visibility)

All Wave 1–3 regressions plus:

| # | Check | How to Verify |
|---|---|---|
| R4.1 | Pod networking not disrupted | Hubble UI deploys as read-only — should not affect traffic |
| R4.2 | Cilium agent healthy | `kubectl -n kube-system exec ds/cilium -- cilium status` |
| R4.3 | Flow collection unaffected | OpenSearch flow count still growing |
| R4.4 | Infra metrics unaffected | Prometheus targets all UP |

## After Wave 5 (Unified UX Go-Live)

All Wave 1–4 regressions plus:

| # | Check | How to Verify |
|---|---|---|
| R5.1 | Homepage loads correctly | `/d/correlation-home` renders in < 5s |
| R5.2 | All specialist dashboards still work | Check each infra dashboard, flow dashboard, health dashboard |
| R5.3 | OpenSearch Dashboards still accessible | Port-forward :5601, query `flows-*` |
| R5.4 | Hubble UI still connected | Port-forward :12000, verify service map |
| R5.5 | No new firing alerts | Check Platform Health dashboard — firing alerts count |

## Full Regression Suite (Post Go-Live)

Run the complete set after all waves are done:

- [ ] R1.1–R1.5 (cluster health)
- [ ] R2.1–R2.5 (telemetry plane)
- [ ] R3.1–R3.5 (flow plane)
- [ ] R4.1–R4.4 (K8s visibility)
- [ ] R5.1–R5.5 (unified UX)
- [ ] T1–T11 from [operator-test-procedures.md](operator-test-procedures.md)
