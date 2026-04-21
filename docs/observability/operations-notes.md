# Operations Notes

Runtime decisions and operational context that affect how the platform is understood but are not architecture changes.

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
