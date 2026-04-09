# Kubernetes Network Visibility Plane

## Architecture Decision

**Cilium and Hubble core are cluster infrastructure, not application workloads.**

This repo manages only the observability-facing layer — specifically Hubble UI deployed in `network-observability`. The Cilium CNI, Cilium operator, Hubble agent, and Hubble Relay are managed in the cluster infrastructure repo (`k8s-lab.git`) via its own ArgoCD workflow.

### Ownership Boundary

| Component | Managed By | Location |
|---|---|---|
| Cilium agent (DaemonSet) | `k8s-lab.git` / cluster ArgoCD | `kube-system` |
| Cilium operator | `k8s-lab.git` / cluster ArgoCD | `kube-system` |
| Cilium CRDs | `k8s-lab.git` / cluster ArgoCD | cluster-wide |
| Hubble (enabled in Cilium config) | `k8s-lab.git` / cluster ArgoCD | `kube-system` |
| Hubble Relay | `k8s-lab.git` / cluster ArgoCD | `kube-system` |
| **Hubble UI** | **`networkanalyzer`** | **`network-observability`** |

### Why This Split

- A Calico → Cilium migration is a cluster-wide CNI swap affecting every pod's networking — not a namespaced application deployment.
- Cilium requires privileged DaemonSets, cluster-admin RBAC, and CRD ownership in `kube-system`.
- Mixing CNI lifecycle with observability application sync would risk accidental network outages from ArgoCD auto-sync.
- The single-Argo-app model for this repo targets `network-observability` namespace — Cilium needs `kube-system`.

## Component Stack

| Component | Version | Purpose | Managed Here |
|---|---|---|---|
| Cilium | 1.19.2 | CNI + eBPF datapath | No — cluster infra |
| Hubble | (bundled with Cilium) | eBPF flow visibility | No — cluster infra |
| Hubble Relay | (bundled with Cilium) | gRPC aggregation of Hubble flows | No — cluster infra |
| Hubble UI | 0.13.3 (via cilium chart 1.19.2) | Web UI for flow investigation | **Yes** |

### Hubble UI Deployment

Hubble UI is deployed using the `cilium/cilium` Helm chart with `hubble.ui.standalone.enabled: true`. This mode:

- Deploys only the UI Deployment + Service
- Disables the Cilium agent, operator, CRDs, and all other components
- Connects to Hubble Relay via gRPC

A Kustomize patch overrides the Relay address from `hubble-relay:80` (same-namespace default) to `hubble-relay.kube-system.svc.cluster.local:80` (cross-namespace).

## Prerequisites

Before this plane is functional:

1. **Cilium must be installed as the cluster CNI** — see [cni-migration-and-rollout.md](cni-migration-and-rollout.md)
2. **Hubble must be enabled in the Cilium configuration** (`hubble.enabled: true`)
3. **Hubble Relay must be deployed** (`hubble.relay.enabled: true`)
4. Hubble Relay must be accessible at `hubble-relay.kube-system.svc.cluster.local:80`

If these prerequisites are not met, Hubble UI will deploy but show no data.

## Supported Deployment Patterns

### Pattern 1: Relay Managed with Cilium (Recommended)

Hubble Relay is deployed as part of the Cilium Helm release in `kube-system`. This repo's Hubble UI connects to it cross-namespace.

```text
kube-system:                    network-observability:
┌─────────────┐                 ┌─────────────┐
│ Cilium Agent│──▶ Hubble ──▶ │ Hubble UI    │
│ (DaemonSet) │                 │ (Deployment) │
└─────────────┘                 └──────┬───────┘
       │                               │
       ▼                               │ gRPC
┌──────────────┐                       │
│ Hubble Relay │◀──────────────────────┘
│ (Deployment) │  hubble-relay.kube-system:80
└──────────────┘
```

### Pattern 2: Relay Exposed as Existing Cluster Service

If Relay is already running and exposed (e.g., via a different mechanism), this repo's Hubble UI consumes it at the same address. No change needed to the values — just ensure the service is reachable.

## Visibility Capabilities

Once Cilium + Hubble are operational, the platform provides:

### Available

- **Pod-to-pod traffic** — source/destination pods, bytes, packets
- **Namespace traffic aggregation** — which namespaces are communicating
- **Service-to-service flows** — traffic between Kubernetes Services
- **DNS visibility** — DNS queries and responses within the cluster
- **L7 HTTP visibility** — HTTP methods, paths, status codes (requires Hubble L7 policy or annotation)
- **L7 gRPC visibility** — gRPC methods and status (same L7 requirement)
- **Egress visibility** — pod traffic to external destinations
- **Network policy verdicts** — allowed/denied flows per CiliumNetworkPolicy

### Requires Additional Configuration

- **L7 visibility** requires either:
  - A CiliumNetworkPolicy with L7 rules on the target namespace/workload, or
  - The `policy.cilium.io/proxy-visibility` pod annotation
  - Without these, traffic is visible at L3/L4 only
- **mTLS decryption** requires service mesh integration (not in scope for this phase)

### Not Available

- **Encrypted payload inspection** — Hubble sees metadata, not payload content
- **Cross-cluster flows** — single cluster only (ClusterMesh would be needed)
- **Historical flow storage** — Hubble UI shows real-time/recent flows; long-term storage requires exporting to the flow analytics plane (Phase 6 correlation)

## Connection to Phase 6 Correlation

This plane provides the Kubernetes-native signal that Phase 6 will correlate with the infra-telemetry and flow-analytics planes:

- **Pod IP → flow analytics** — pod IPs from Hubble correlate with src/dst IPs in OpenSearch flow records
- **K8s node → Proxmox VM** — node names map to VM names in the infra-telemetry plane
- **DNS queries** — cluster DNS from Hubble supplements external DNS enrichment in the flow plane
- **Service → endpoint** — Kubernetes Service relationships provide the "path-style" entity chain

## Self-Health

- Hubble UI Deployment has readiness/liveness probes via the Helm chart
- Relay health is visible in the Cilium/Hubble status (managed in cluster infra)
- Hubble UI shows a connection status indicator for its Relay connection
