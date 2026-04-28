# Operations Notes

Runtime decisions and operational context that affect how the platform is understood but are not architecture changes.

## 2026-04-27 — Wave 5c Pre-Soak: K8s Enrichment Runbook

k8s-ip-exporter CronJob is now running on a 30-minute schedule (`suspend: false` committed at Wave 5c soak start). ArgoCD is configured to leave runtime-written ConfigMap data intact — `ignoreDifferences` on `.data` plus `RespectIgnoreDifferences=true` prevents header-only placeholder data from being re-applied on sync. This was confirmed working: ConfigMap data survived the 2026-04-27 sync at 05:04:45Z.

### How the ConfigMap data is protected from ArgoCD sync resets

**Background:** ArgoCD v3.3.6 with `ServerSideApply=true` does not honour
`RespectIgnoreDifferences=true` for ConfigMap `.data` — it applies the header-only
placeholder from git on every sync, wiping runtime data.

**Fix:** A PostSync hook Job (`k8s-ip-exporter-postsync`) runs immediately after
every ArgoCD sync. It calls the same exporter script as the CronJob and
repopulates the ConfigMap with live cluster data, then restarts `flow-collector`
if the data changed. The whole cycle takes ~30 seconds.

The `ignoreDifferences` entry for `.data` still works for **comparison** (ArgoCD sees
the ConfigMap as Synced after the hook writes live data, so `selfHeal` does not
trigger a second sync). The protection chain: sync resets data → PostSync hook
repopulates → ArgoCD sees Synced → stable.

```bash
# Confirm the PostSync hook Job completed successfully after last sync
kubectl get job k8s-ip-exporter-postsync -n network-observability
kubectl logs -n network-observability job/k8s-ip-exporter-postsync --tail=5
```

### Check lookup table row counts

```bash
# pods.csv: expect > 1 line (header + data rows)
kubectl get configmap k8s-lookup-tables -n network-observability \
  -o jsonpath='{.data.pods\.csv}' | wc -l

# services.csv
kubectl get configmap k8s-lookup-tables -n network-observability \
  -o jsonpath='{.data.services\.csv}' | wc -l

# nodes.csv
kubectl get configmap k8s-lookup-tables -n network-observability \
  -o jsonpath='{.data.nodes\.csv}' | wc -l
```

Healthy baseline: ~80 pods, ~38 services, ~5 nodes in this homelab cluster.

### Trigger an on-demand refresh

```bash
kubectl create job --from=cronjob/k8s-ip-exporter \
  manual-refresh-$(date +%s) -n network-observability

# Watch job logs (wait ~10s for pod to start)
kubectl logs -n network-observability \
  -l app.kubernetes.io/name=k8s-ip-exporter --since=2m -f
```

The job exits 0 with "Done." on success. If cluster state is unchanged it exits 0
with "Done (no-op)." — no restart is triggered.

### Verify fresh K8s enrichment fields in OpenSearch

```bash
# Port-forward (if not already running)
kubectl port-forward -n network-observability \
  svc/opensearch-cluster-master 9200:9200 &

# Flows from last 1h with src_k8s_namespace enrichment
curl -s 'localhost:9200/flows-*/_count?q=_exists_:src_k8s_namespace'

# Flows with node and workload fields
curl -s 'localhost:9200/flows-*/_count?q=_exists_:src_k8s_node'
```

A non-zero count on `src_k8s_namespace` confirms enriched flows are being indexed.
If the count is zero, run an on-demand refresh and restart flow-collector.

### Clean stale non-running flow-collector pods

Completed/Error pods from rolling restarts accumulate over time. These are
harmless but add noise to `kubectl get pods` output.

```bash
# List all flow-collector pods
kubectl get pods -n network-observability -l app.kubernetes.io/name=flow-collector

# Delete only non-Running pods
kubectl get pods -n network-observability \
  -l app.kubernetes.io/name=flow-collector \
  --field-selector='status.phase!=Running' -o name \
  | xargs --no-run-if-empty kubectl delete -n network-observability
```

**Do NOT delete the active 2/2 Running flow-collector pod** unless intentionally
restarting it. Deleting the Running pod restarts it and causes a ~30-second gap
in flow collection. Use `kubectl rollout restart` instead if a restart is needed.

---

## 2026-04-25 — Wave 5: Network Observability Grafana Access

The `network-observability` Grafana is exposed via MetalLB at a dedicated static IP.

**Access:**

- URL: `http://netgrafana.vgriz.com`
- IP: `172.18.1.212` (MetalLB, static)
- Port: `80`

**DNS:** `netgrafana.vgriz.com` resolves to `172.18.1.212`. This is separate from
`grafana.vgriz.com` (`172.18.1.201`), which is the `monitoring` namespace Grafana
from `kube-prometheus-stack`. Both remain independent — do not point them at the
same IP.

**How it is configured:** `service.type: LoadBalancer` and
`service.loadBalancerIP: "172.18.1.212"` in
`platform/base/infra-telemetry/grafana-values.yaml`. The MetalLB annotation
(`metallb.universe.tf/loadBalancerIPs`) is intentionally absent — MetalLB rejects
services that set both `spec.loadBalancerIP` and the annotation simultaneously.

**Decision owner:** Mike Beil (operator), Wave 5 rollout.

---

## 2026-04-26 — Wave 5b: Home Dashboard as Grafana Front Door

The `Network Observability — Home` dashboard is the primary entry point for the
`network-observability` Grafana instance.

**Dashboard:**

- Title: `Network Observability — Home`
- UID: `net-obs-home`
- URL: `http://netgrafana.vgriz.com/d/net-obs-home`

**Navigation links (all validated at Gate 5b):**

- Flow — Traffic Mix
- Flow — K8s Context
- Flow — Top Talkers
- Flow — Destination Analysis
- Proxmox Node & VM Network
- Switch Interface Utilization
- UniFi AP & WLAN Clients

**CNI:** Calico (not Cilium — corrected in hotfix `e63b082`).

**K8s enrichment status displayed:** lookup tables populated; CronJob suspended;
refreshes on demand.

---

## 2026-04-26 — Gate 5a Operational Fixes

The following issues were discovered and fixed during Gate 5a validation.
They are documented here as stable operational facts.

### Grafana Deployment Must Use Recreate Strategy (RWO PVC)

Grafana uses a `rook-ceph-block` ReadWriteOnce PVC. The Helm chart default
rolling update strategy attempts to schedule a new pod before the old one
terminates. If the two pods land on different nodes, the new pod cannot mount
the PVC (Multi-Attach error) and the rollout stalls indefinitely.

**Fix:** `deploymentStrategy: type: Recreate` in
`platform/base/infra-telemetry/grafana-values.yaml`. A Kustomize JSON 6902
patch (`platform/overlays/lab/patches/grafana-strategy-recreate.yaml`) also
sets `rollingUpdate: null` explicitly so that ArgoCD server-side apply clears
the field from the live object rather than leaving a conflicting value.

**Rule:** Any stateful Grafana deployment backed by a RWO PVC must use
`Recreate`. Do not revert to `RollingUpdate` without replacing the PVC with
a ReadWriteMany storage class.

### Prometheus Service URL Uses Port 80

The Prometheus service in the `monitoring` namespace exposes port 80, not 9090.
Port 9090 is the Prometheus container port but is not the Kubernetes service port.
Grafana datasource URLs pointing to port 9090 silently fail or return connection
refused.

**Correct URL:**
`http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090`
is **wrong**. The service port is **80**:

```text
http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:80
```

The value is configured as the `url` in the `Prometheus GitOps` datasource
entry in `platform/base/infra-telemetry/grafana-values.yaml`.

### k8s-ip-exporter Lookup Tables Must Remain Populated at Runtime

The `k8s-ip-exporter` CronJob writes live pod/service/node lookup CSV files
into the `k8s-lookup-tables` ConfigMap. These are mounted into `flow-collector`
(Vector). If the CronJob has not run, or if the ConfigMap has been reverted
by ArgoCD to its placeholder headers (see `ignoreDifferences` in
`platform/apps/network-observability.yaml`), all internal flows will classify
as `internal-unknown`.

**Post-sync required steps** after any ArgoCD sync that restarts `flow-collector`:

```bash
# 1. Trigger a CronJob run to repopulate lookup tables
kubectl create job --from=cronjob/k8s-ip-exporter manual-refresh -n network-observability

# 2. Restart flow-collector after tables are populated (Vector does not live-reload)
kubectl rollout restart deploy/flow-collector -n network-observability
```

Do not treat a rising `internal-unknown` rate as a bug without first confirming
the CronJob has run successfully since the last `flow-collector` restart.

---

## 2026-04-25 — Grafana: Persisted Datasource UID Conflict

The lab Grafana instance had an existing persisted datasource named **Prometheus**
with an autogenerated UID (`PBFA97CFB590B2093`). Provisioning a new datasource with
the same name (`Prometheus`) and an explicit `uid: prometheus` causes a CrashLoopBackOff
at startup: Grafana refuses to overwrite a persisted datasource with a conflicting UID.

**Resolution:** The GitOps-provisioned datasource is named **Prometheus GitOps** with
`uid: prometheus`. The original persisted datasource retains its name and autogenerated
UID and is left untouched.

**Dashboard convention:** All dashboards in this platform must reference UID `prometheus`
(i.e., the GitOps datasource), not the persisted datasource name or its autogenerated UID.
This ensures dashboards work after a PVC reset or fresh cluster deploy.

---

## 2026-04-25 — Grafana: Datasource Provisioning Requires Pod Restart

The Prometheus datasource is provisioned from a ConfigMap on Grafana startup. If the
pod was running before the `uid: prometheus` field was added to
`grafana-values.yaml`, it will not pick up the UID until restarted. Symptom:
"Datasource not found" errors on the Proxmox, Switch, and UniFi dashboards.

After an ArgoCD sync that changes datasource configuration:

```bash
kubectl rollout restart deploy/grafana -n network-observability
```

---

## 2026-04-25 — K8s Enrichment: Post-Sync Operational Steps

After an ArgoCD sync of the network-observability app, Wave 3b K8s enrichment
requires two steps to be fully active:

1. **Refresh lookup tables** — trigger the k8s-ip-exporter CronJob manually or wait
   for its next scheduled run:

   ```bash
   kubectl create job --from=cronjob/k8s-ip-exporter manual-refresh -n network-observability
   ```

2. **Restart flow-collector** — Vector does not live-reload enrichment tables:

   ```bash
   kubectl rollout restart deploy/flow-collector -n network-observability
   ```

### Validation commands

```bash
# 1. Confirm lookup tables are populated (expect > 1 line — header + data rows)
kubectl get configmap k8s-lookup-tables -n network-observability \
  -o jsonpath='{.data.pods\.csv}' | wc -l

# 2. Confirm flow-collector is healthy
kubectl rollout status deploy/flow-collector -n network-observability

# 3. Verify recent flows have K8s namespace enrichment (via OpenSearch)
#    Port-forward first: kubectl port-forward -n network-observability svc/opensearch-cluster-master 9200:9200
curl -s 'localhost:9200/flows-*/_count?q=_exists_:src_k8s_namespace'
curl -s 'localhost:9200/flows-*/_count?q=_exists_:src_k8s_node'
```

K8s Context dashboard panels are reliable only after the lookup tables are populated
and flow-collector has been restarted. Until then, most flows will appear as
`internal-unknown` or `external`.

---

## 2026-04-20 — Wave 3b: VRL Hotfix History and Operational Decisions

### VRL function name: `ip_cidr_contains` not `cidr_contains`

Vector 0.45.0 (`timberio/vector:0.45.0-distroless-static`) exposes the CIDR check
function as `ip_cidr_contains`. The name `cidr_contains` does not exist in this
build and causes a hard startup failure (undefined function). The VRL documentation
and function name diverge between Vector builds.

**If upgrading Vector:** always validate CIDR function name in a test environment
before deploying. The function name is build-specific, not stable across Vector
versions.

### `ip_cidr_contains` is fallible in Vector 0.45.0

`ip_cidr_contains` returns `Result<bool>`, not `bool`. Using it in a plain
assignment triggers Vector startup error E103 (unhandled fallible). The required
pattern is:

```vrl
var, err = ip_cidr_contains("cidr", ip_string)
if err != null { var = false }
```

This must be applied to every call site. A `?? false` suffix does NOT work for
infallible coercion on this function — the err guard is required.

### ArgoCD `ignoreDifferences` required for two runtime-written resources

Two resources in this platform are written at runtime by the CronJob and must be
excluded from ArgoCD `selfHeal` reconciliation:

1. **`flow-collector` Deployment** — `kubectl.kubernetes.io/restartedAt` annotation
   (written by CronJob rolling restart). Without `ignoreDifferences`, ArgoCD reverts
   the annotation within seconds, which does not cause a functional failure but
   produces continuous drift noise.

2. **`k8s-lookup-tables` ConfigMap** — `.data` field (CronJob PATCHes live CSV
   content; Git holds header-only placeholders). Without `ignoreDifferences`, ArgoCD
   reverts the ConfigMap to placeholder headers on every sync cycle, causing
   `internal-unknown` classification for all flows despite a successful CronJob run.
   This is a silent data-quality failure — flow-collector remains healthy.

Both `ignoreDifferences` entries are in `platform/apps/network-observability.yaml`.

### Distroless container: no exec access

The `timberio/vector:0.45.0-distroless-static` image has no shell. `kubectl exec`
is not possible. VRL errors are only visible via pod logs at startup. To debug VRL:

1. Read Vector startup logs: `kubectl logs -n network-observability deploy/flow-collector`
2. Errors include the VRL source line and error code (e.g., E103).
3. For interactive debugging, use a non-distroless Vector image in a test pod.

**Decision owner:** Mike Beil (operator), Wave 3b rollout. Accepted as operational
constraint; mitigation tracked in backlog.

## 2026-04-11 — Legacy Monitoring Cleanup

**Context:** During Wave 1 deployment, the existing `monitoring` namespace Grafana (from `kube-prometheus-stack`) was found to be unhealthy (0/1 pods). The dedicated `network-observability` Grafana was deployed as a standalone instance and is now the active platform UI.

**Action taken:** The legacy `monitoring` Grafana and Loki were operationally isolated during a runtime cleanup. This was a manual ops action to reduce confusion and resource contention, not a Wave 1 architecture change.

**Current state:**

- `network-observability` Grafana is the active platform UI
- `network-observability` Grafana queries the existing Prometheus in `monitoring` namespace
- The existing Prometheus in `monitoring` remains the scrape backend and is healthy
- Legacy Grafana/Loki in `monitoring` are not actively used by this platform

**Impact on this repo:**

- None. This repo never owned the legacy Grafana or Loki.
- The existing Prometheus service (`kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090`) remains the datasource for the dedicated Grafana.
- If the legacy monitoring stack is fully removed in the future, only the Prometheus instance must be preserved (or replaced with an equivalent scrape backend).

**Decision owner:** Mike Beil (operator), runtime ops action.

## 2026-04-12 — Softflowd Service Pattern: Explicit Unit Required

**Context:** During Wave 2 host rollout, several Proxmox hosts had both the
package-installed default softflowd paths and a custom `softflowd-netflow.service`
unit. Running both simultaneously caused double-exporting or silent conflicts.

**Why legacy paths are unsafe:**

The package-installed `softflowd.service` and `softflowd@default.service` units
read from `/etc/default/softflowd` but are not controlled by this platform's
deployment. If the package is upgraded or the default config is reset, flows may
revert to the old target (`172.18.1.207:2055`) or stop entirely without operator
notice. Masking these units prevents accidental re-activation.

**Supported pattern — explicit custom unit:**

```bash
# 1. Stop and mask the package/default paths
sudo systemctl stop softflowd || true
sudo systemctl disable softflowd || true
sudo systemctl mask softflowd || true
sudo systemctl stop softflowd@default || true

# 2. Activate the explicit platform unit
sudo systemctl start softflowd-netflow.service
sudo systemctl enable softflowd-netflow.service
```

The `softflowd-netflow.service` unit is the sole authoritative exporter for this
platform. It is not managed by the Debian/Ubuntu package lifecycle.

**Interface autodetect lesson learned:**

Different Proxmox hosts use different interface names (`eno1`, `eth0`, `bond0`).
Hardcoding the interface name in the service unit template caused silent failures
on hosts where the interface name differed from the expected value.

The correct pattern is to derive the interface from the existing
`/etc/default/softflowd` config on each host:

```bash
IFACE=$(grep -oP '(?<=-i )\S+' /etc/default/softflowd)
```

This extracts the `-i <interface>` argument from the host's own config, which was
set correctly at initial softflowd setup and reflects the actual primary interface.

**Decision owner:** Mike Beil (operator), Wave 2 rollout.
