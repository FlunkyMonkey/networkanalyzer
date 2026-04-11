# Operations Notes

Runtime decisions and operational context that affect how the platform is understood but are not architecture changes.

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
