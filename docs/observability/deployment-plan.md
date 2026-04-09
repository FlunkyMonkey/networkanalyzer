# Deployment Plan

This document defines the phased rollout model for the network observability platform. Each wave is independently deployable with explicit verification and rollback gates.

**This plan is executed in Phase 8.** Phase 7 prepares and validates the plan; Phase 8 executes it.

## Rollout Waves

### Wave 0 — Preflight and Readiness

**Purpose:** Verify all prerequisites are met before any deployment.

**Prerequisites:**

- Git repo renders clean: `kustomize build --enable-helm platform/overlays/lab`
- Markdown lint passes: `markdownlint docs/**/*.md`
- Kubernetes cluster is healthy and accessible
- ArgoCD is running and the private repo is registered
- Cilium CNI is installed with Hubble and Relay enabled (managed in `k8s-lab.git`)
- `network-observability` namespace can be created
- Ceph StorageClass `ceph-block` is available
- Network connectivity verified: MikroTik:161/udp, UniFi:8443/tcp, Proxmox:8006/tcp

**Execution order:**

1. Run `kustomize build --enable-helm platform/overlays/lab` — verify clean render
1. Verify cluster access: `kubectl get nodes`
1. Verify ArgoCD: `argocd repo list | grep networkanalyzer`
1. Verify Cilium: `kubectl -n kube-system exec ds/cilium -- cilium status`
1. Verify Hubble Relay: `kubectl -n kube-system get svc hubble-relay`
1. Verify StorageClass: `kubectl get sc ceph-block`
1. Create namespace: `kubectl create namespace network-observability`
1. Create all required Secrets (see [secrets-and-bootstrap.md](secrets-and-bootstrap.md))

**Verification:**

- All commands succeed without errors
- Secrets are created and verified: `kubectl get secrets -n network-observability`

**Stop conditions:** Any prerequisite fails. Do not proceed to Wave 1.

**Rollback:** Delete namespace: `kubectl delete namespace network-observability`

---

### Wave 1 — Telemetry Plane Only

**Purpose:** Deploy Prometheus, Grafana, Alertmanager, and all infrastructure exporters. Verify metrics collection before adding flow analytics.

**Prerequisites:** Wave 0 passed.

**Execution order:**

1. Apply the ArgoCD Application: `kubectl apply -f platform/apps/network-observability.yaml`
1. Wait for infra-telemetry pods to become Ready
1. Verify Prometheus is scraping targets
1. Verify Grafana is accessible and dashboards are loaded
1. Verify SNMP exporter is collecting MikroTik interface metrics
1. Verify UnPoller is collecting WLAN client metrics
1. Verify Proxmox exporter is collecting VM metrics

**Verification:**

- `kubectl get pods -n network-observability` — all pods Running/Ready
- Prometheus Targets page shows all exporters as UP
- Grafana dashboards show live data: interface utilization, WiFi clients, Proxmox VMs

**Stop conditions:** Any exporter fails to scrape. Prometheus cannot reach targets. Grafana does not load.

**Rollback:** Delete the ArgoCD Application: `kubectl delete -f platform/apps/network-observability.yaml`. Then delete PVCs if needed.

---

### Wave 2 — Flow Plane Infrastructure

**Purpose:** Deploy OpenSearch, OpenSearch Dashboards, and the GoFlow2+Vector collector. Verify the stack is healthy before repointing softflowd.

**Prerequisites:** Wave 1 passed. All infra-telemetry metrics are flowing.

**Execution order:**

1. ArgoCD sync deploys flow-analytics components automatically (they are part of the same Application)
1. Wait for OpenSearch StatefulSet to become Ready
1. Wait for OpenSearch Dashboards pod to become Ready
1. Wait for flow-collector pod to become Ready
1. Load the OpenSearch index template (see [flow-ingest-and-cutover.md](flow-ingest-and-cutover.md))
1. Load the ISM retention policy
1. Import OpenSearch Dashboards saved objects
1. Verify the flow-collector Service has an external IP assigned by MetalLB

**Verification:**

- OpenSearch cluster health is green: `curl http://localhost:9200/_cluster/health` (via port-forward)
- Flow collector pod shows both containers (goflow2 + vector) Running
- `flow-collector` Service has an EXTERNAL-IP
- OpenSearch index template exists
- ISM policy is active

**Stop conditions:** OpenSearch fails to start. PVC does not bind. MetalLB does not assign an IP.

**Rollback:** The flow-analytics components are part of the single ArgoCD app. To isolate: set `resources: []` in `platform/base/flow-analytics/kustomization.yaml`, commit, and let ArgoCD prune. Or scale down: `kubectl scale deploy/flow-collector --replicas=0 -n network-observability`.

---

### Wave 3 — Softflowd Cutover

**Purpose:** Repoint existing softflowd exporters from their current destination to the in-cluster flow collector. This is the first live data cutover.

**Prerequisites:** Wave 2 passed. Flow collector is Running with an external IP. No flows are being received yet.

**Execution order:**

1. Record the current softflowd destination on each Proxmox host (for rollback)
1. Repoint softflowd on **one** Proxmox host first (canary)
1. Verify flows appear in GoFlow2: `kubectl exec deploy/flow-collector -c goflow2 -- tail -5 /flows/flows.jsonl`
1. Verify flows appear in OpenSearch: `curl http://localhost:9200/flows-*/_count`
1. If canary is healthy, repoint remaining Proxmox hosts
1. Verify all hosts are exporting flows
1. Verify OpenSearch Dashboards show flow data

**Verification:**

- GoFlow2 is writing JSON records
- Vector health endpoint returns 200
- OpenSearch has flow indices with growing document counts
- OpenSearch Dashboards can query flows
- Flow data includes expected fields (src_ip, dst_ip, bytes, protocol)
- GeoIP enrichment is present if MaxMind key was configured

**Stop conditions:** GoFlow2 not receiving flows (check firewall/routing). Vector failing to index (check OpenSearch connectivity). Corrupted flow data.

**Rollback:** Repoint softflowd back to the original destination on each Proxmox host. Restart softflowd. In-cluster collector stops receiving but remains deployed.

---

### Wave 4 — K8s Visibility Activation

**Purpose:** Verify Hubble UI connects to Hubble Relay and shows Kubernetes flow data.

**Prerequisites:** Wave 1 passed. Cilium + Hubble are running in the cluster (prerequisite from Wave 0).

**Execution order:**

1. Verify Hubble UI pod is Running: `kubectl get pods -n network-observability -l app.kubernetes.io/name=hubble-ui`
1. Port-forward to Hubble UI: `kubectl port-forward -n network-observability svc/hubble-ui 12000:80`
1. Open `http://localhost:12000`
1. Verify Hubble UI shows pod-to-pod flows
1. Filter by namespace — verify namespace-level aggregation
1. Check for DNS visibility

**Verification:**

- Hubble UI loads and shows a service map
- Pod-to-pod flows are visible
- Namespace filtering works
- DNS queries are visible (if Hubble DNS monitoring is enabled in Cilium)

**Stop conditions:** Hubble UI shows "Relay unavailable" — check that Hubble Relay is running in kube-system and reachable.

**Rollback:** Hubble UI is read-only and has no impact on cluster networking. Scale to 0 if needed: `kubectl scale deploy/hubble-ui --replicas=0 -n network-observability`.

---

### Wave 5 — Unified UX Go-Live

**Purpose:** Verify the full platform experience: homepage, drill-downs, playbooks, and cross-plane correlation.

**Prerequisites:** Waves 1–4 passed. All data sources are flowing.

**Execution order:**

1. Access Grafana: `kubectl port-forward -n network-observability svc/prometheus-grafana 3000:80`
1. Navigate to the homepage dashboard (`/d/correlation-home`)
1. Verify homepage card order: top talkers, destinations, WiFi client bandwidth
1. Verify the freshness bar shows healthy status
1. Click a source IP in Top Talkers — verify it opens Entity Investigation
1. Enter an IP in Entity Investigation — verify flow tables populate
1. Navigate to Investigation Playbooks — verify all 6 playbooks are present
1. Navigate to Platform Health — verify scrape targets are UP
1. Click the OpenSearch Dashboards link — verify it opens and shows flow data
1. Click the Hubble UI link — verify it opens and shows K8s flows
1. Test the "slow internet" playbook end-to-end

**Verification:**

- Homepage loads in under 5 seconds
- All 3 locked cards show live data
- Cross-plane drill-down from IP → flows → K8s works
- Playbook links resolve to correct dashboards
- Health dashboard shows no DOWN targets
- OpenSearch and Hubble UI are reachable from Grafana links

**Stop conditions:** Homepage does not load. OpenSearch datasource errors. Critical drill-down paths are broken.

**Rollback:** UX issues are non-destructive. Fix and re-sync via ArgoCD. No data loss risk from this wave.
