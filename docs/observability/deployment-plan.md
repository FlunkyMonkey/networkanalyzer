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
- Rook-Ceph StorageClass is available (`rook-ceph-block` is the cluster default)
- Network connectivity verified: MikroTik:161/udp, UniFi:8443/tcp, Proxmox:8006/tcp

**Note:** Cilium/Hubble are NOT required for Wave 0. They are a prerequisite for Wave 4 (K8s visibility) only. Waves 1–3 can proceed without Cilium.

**Already completed:**

- `network-observability` namespace created
- `grafana-admin-credentials` Secret created
- `unpoller-credentials` Secret created
- `proxmox-credentials` Secret created

**Execution order:**

1. Run `kustomize build --enable-helm platform/overlays/lab` — verify clean render
1. Verify cluster access: `kubectl get nodes`
1. Verify ArgoCD: `argocd repo list | grep networkanalyzer`
1. Verify StorageClass: `kubectl get sc rook-ceph-block`
1. Verify namespace exists: `kubectl get ns network-observability`
1. Verify Secrets exist: `kubectl get secrets -n network-observability`
1. Create `geoip-credentials` Secret if MaxMind key is available (optional)

**Verification:**

- All commands succeed without errors
- Secrets are created and verified: `kubectl get secrets -n network-observability`

**Stop conditions:** Any prerequisite fails. Do not proceed to Wave 1.

**Rollback:** Delete namespace: `kubectl delete namespace network-observability`

---

### Wave 1 — Telemetry Plane Only

**Purpose:** Deploy standalone Grafana and infrastructure exporters (SNMP, UnPoller, Proxmox). Scraping is handled by the existing cluster Prometheus in the `monitoring` namespace.

**Prerequisites:** Wave 0 passed. Lab overlay already includes only namespace + infra-telemetry (default state). Existing Prometheus must be healthy and watching all namespaces for ServiceMonitors labeled `release: kube-prometheus-stack`.

**Wave enablement:** No repo change needed — the lab overlay ships with Wave 1 enabled. See [wave-enablement-model.md](wave-enablement-model.md).

**Execution order:**

1. Apply the ArgoCD Application: `kubectl apply -f platform/apps/network-observability.yaml`
1. Wait for all pods to become Ready (Grafana, SNMP exporter, UnPoller, Proxmox exporter)
1. Verify existing Prometheus has picked up the new ServiceMonitors: check Prometheus Targets page in the `monitoring` namespace
1. Verify Grafana is accessible and can query metrics from existing Prometheus
1. Verify SNMP exporter is collecting MikroTik interface metrics
1. Verify UnPoller is collecting WLAN client metrics
1. Verify Proxmox exporter is collecting VM metrics

**Verification:**

- `kubectl get pods -n network-observability` — all pods Running/Ready
- Existing Prometheus Targets page shows 3 new targets (snmp, unpoller, proxmox) as UP
- Grafana dashboards show live data: interface utilization, WiFi clients, Proxmox VMs
- Grafana datasource test passes for Prometheus connection

**Retention note:** Existing Prometheus retains 7 days. The 30-day retention objective is not yet satisfied. This is accepted for Wave 1. Address in Phase 9.

**Stop conditions:** Exporters not appearing in existing Prometheus targets (check ServiceMonitor labels). Grafana cannot connect to Prometheus. Exporters crash-looping.

**Rollback:** Delete the ArgoCD Application: `kubectl delete -f platform/apps/network-observability.yaml`. Then delete PVCs if needed. Existing Prometheus is unaffected.

---

### Wave 2 — Flow Plane Infrastructure

**Purpose:** Deploy OpenSearch and the GoFlow2+Vector collector. Verify the stack is healthy before repointing softflowd. Grafana is provisioned with the OpenSearch datasource and flow dashboards as part of this wave.

**Note:** OpenSearch Dashboards is NOT deployed. Grafana is the sole UI surface. Flow investigation is done via the `flow-top-talkers`, `flow-destinations`, and `flow-traffic-mix` dashboards in the existing platform Grafana.

**Prerequisites:** Wave 1 passed. All infra-telemetry metrics are flowing.

**Wave enablement:** Uncomment `../../base/flow-analytics` in `platform/overlays/lab/kustomization.yaml`, commit, and push. ArgoCD syncs the new plane. See [wave-enablement-model.md](wave-enablement-model.md).

**Execution order:**

1. Enable flow-analytics in the lab overlay (repo commit), then ArgoCD sync
1. Wait for OpenSearch StatefulSet to become Ready (3–7 min — JVM startup)
1. Wait for flow-collector pod to become Ready (both goflow2 + vector containers)
1. Load the OpenSearch index template (see [flow-ingest-and-cutover.md](flow-ingest-and-cutover.md))
1. Load the ISM retention policy (`flow-retention-14d`)
1. Verify the flow-collector Service has an external IP assigned by MetalLB
1. Verify Grafana has picked up the OpenSearch datasource and flow dashboards

**Verification:**

- OpenSearch cluster health is green: `curl http://localhost:9200/_cluster/health` (via port-forward)
- Flow collector pod shows both containers (goflow2 + vector) Running
- `flow-collector` Service has an EXTERNAL-IP
- OpenSearch index template exists: `curl http://localhost:9200/_index_template/flows`
- ISM policy `flow-retention-14d` is active
- Grafana: Configuration → Data Sources → "OpenSearch Flows" → Test passes
- Grafana: Dashboards → Browse shows `flow-top-talkers`, `flow-destinations`, `flow-traffic-mix`

**Security stance (canary stage):** OpenSearch security plugin is disabled (`plugins.security.disabled: true`). Access is unauthenticated in-cluster HTTP. This is acceptable for the canary stage only. Hardening (TLS, authentication) is required before any production promotion. Documented as a future hardening item in [flow-analytics-plane.md](flow-analytics-plane.md).

**Stop conditions:** OpenSearch fails to start. PVC does not bind. MetalLB does not assign an IP. Grafana cannot connect to OpenSearch.

**Rollback:** Re-comment the `../../base/flow-analytics` line in `platform/overlays/lab/kustomization.yaml`, commit, and push. ArgoCD auto-prune removes flow-analytics resources including the Grafana flow ConfigMaps. Grafana reverts to Prometheus-only datasource automatically.

---

### Wave 3 — Softflowd Canary Cutover

**Purpose:** Repoint softflowd exporters from the legacy collector to the in-cluster flow collector. This is the first live data cutover. Uses a canary-first approach.

**Legacy rollback target:** `172.18.1.207:2055/UDP`
**Canary host:** prox5 (already exporting NetFlow v9)

**Prerequisites:** Wave 2 passed. Flow collector is Running with an external IP. Grafana flow dashboards are loading. See full runbook: [wave2-canary-cutover.md](wave2-canary-cutover.md).

**Execution order:**

1. Record the current softflowd destination on each Proxmox host (for rollback)
1. Repoint softflowd on **prox5 only** (canary)
1. Verify flows appear in GoFlow2: `kubectl exec deploy/flow-collector -c goflow2 -- tail -5 /flows/flows.jsonl`
1. Verify flows appear in OpenSearch: `curl http://localhost:9200/flows-*/_count`
1. Verify canary soak for 15+ minutes — doc count growing, no Vector errors
1. Verify Grafana "Flow — Top Talkers" dashboard shows data
1. If canary is healthy, repoint remaining Proxmox hosts
1. Verify all hosts are exporting flows (one exporter bucket per host in OpenSearch)

**Verification:**

- GoFlow2 is writing JSON records with correct `exporter` field (= prox5 IP)
- Vector health endpoint returns 200
- OpenSearch has flow indices with growing document counts
- Flow data includes expected fields (src_ip, dst_ip, bytes, protocol_name, exporter)
- GeoIP enrichment is present if MaxMind key was configured
- Grafana "Flow — Top Talkers" table shows real IP data

**Stop conditions:** GoFlow2 not receiving flows (check firewall/routing from prox5 to FLOW_COLLECTOR_IP:2055/UDP). Vector failing to index (check OpenSearch connectivity). Corrupted flow data.

**Rollback:** Restore `/etc/default/softflowd.bak` on prox5, restart softflowd. Legacy collector at `172.18.1.207:2055` resumes immediately. In-cluster collector stops receiving but remains deployed.

---

### Wave 4 — K8s Visibility Activation

**Purpose:** Verify Hubble UI connects to Hubble Relay and shows Kubernetes flow data.

**Prerequisites:**

- Wave 1 passed
- Cilium CNI is installed with Hubble and Relay enabled (managed in `k8s-lab.git`)
- Hubble Relay is running and reachable at `hubble-relay.kube-system.svc.cluster.local:80`

**Note:** This is the first wave that requires Cilium. If Cilium migration has not been completed yet, skip Wave 4 and proceed to Wave 5. K8s visibility can be activated later independently.

**Wave enablement:** Uncomment `../../base/k8s-visibility` in `platform/overlays/lab/kustomization.yaml`, commit, and push. See [wave-enablement-model.md](wave-enablement-model.md).

**Pre-wave verification:**

1. Verify Cilium is running: `kubectl -n kube-system exec ds/cilium -- cilium status`
1. Verify Hubble Relay: `kubectl -n kube-system get deploy hubble-relay`

**Execution order:**

1. Enable k8s-visibility in the lab overlay (repo commit), then ArgoCD sync
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

**Prerequisites:** Waves 1–3 passed. Wave 4 passed or deferred (Cilium not required for UX go-live). All infra and flow data sources are flowing.

**Wave enablement:** Uncomment `../../base/correlation-ux` in `platform/overlays/lab/kustomization.yaml`, commit, and push. See [wave-enablement-model.md](wave-enablement-model.md).

**Execution order:**

1. Enable correlation-ux in the lab overlay (repo commit), then ArgoCD sync
1. Access Grafana: `kubectl port-forward -n network-observability svc/grafana 3000:80`
1. Navigate to the homepage dashboard (`/d/correlation-home`)
1. Verify homepage card order: top talkers, destinations, WiFi client bandwidth
1. Verify the freshness bar shows healthy status
1. Click a source IP in Top Talkers — verify it opens Entity Investigation
1. Enter an IP in Entity Investigation — verify flow tables populate
1. Navigate to Investigation Playbooks — verify all 6 playbooks are present
1. Navigate to Platform Health — verify scrape targets are UP
1. Click "View in Flow — Top Talkers" link — verify Grafana flow dashboard opens with data
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
