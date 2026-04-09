# CNI Migration and Rollout

## Scope

This document describes the Calico → Cilium CNI migration required as a prerequisite for the Kubernetes network visibility plane.

**This migration is managed in `k8s-lab.git` (cluster infrastructure), not in this repo.**

This repo (`networkanalyzer`) only deploys Hubble UI. The Cilium agent, operator, CRDs, and Hubble Relay are owned by the cluster infrastructure workflow.

## Migration Overview

| Step | Description | Risk | Managed By |
|---|---|---|---|
| 1 | Plan and test in non-production first | Low | Operator |
| 2 | Remove Calico | **High** — causes network outage | `k8s-lab.git` |
| 3 | Install Cilium with Hubble enabled | **High** — restores networking | `k8s-lab.git` |
| 4 | Verify pod networking restored | Medium | Operator |
| 5 | Deploy Hubble UI (this repo) | Low | `networkanalyzer` ArgoCD |

## Recommended Cilium Values for k8s-lab.git

When installing Cilium in the cluster infrastructure repo, include these settings to enable Hubble with the visibility this platform needs:

```yaml
# Cilium Helm values for k8s-lab.git
# These enable the Hubble observability layer consumed by the networkanalyzer platform.

hubble:
  enabled: true

  relay:
    enabled: true

  metrics:
    enabled:
      - dns
      - drop
      - tcp
      - flow
      - icmp
      - http

  # Export Hubble metrics to Prometheus (scraped by the infra-telemetry plane).
  prometheus:
    enabled: true
    serviceMonitor:
      enabled: true
```

## Rollout Order

### Pre-Migration Checklist

1. Back up current cluster state:

   ```bash
   kubectl get nodes -o wide
   kubectl get pods --all-namespaces -o wide > pods-before-migration.txt
   kubectl get networkpolicies --all-namespaces -o yaml > netpol-backup.yaml
   ```

1. Verify cluster health:

   ```bash
   kubectl get nodes
   kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded
   ```

1. Schedule a maintenance window — pod networking will be disrupted during migration.

1. Notify users/services that depend on the cluster.

### Migration Steps (Managed in k8s-lab.git)

1. **Remove Calico:**

   ```bash
   # If Calico was installed via manifest:
   kubectl delete -f https://docs.projectcalico.org/manifests/calico.yaml
   # If via Helm:
   helm uninstall calico -n kube-system
   # If via operator:
   kubectl delete installation default
   ```

   **Warning:** This causes immediate network disruption. All pods lose connectivity until Cilium is installed.

1. **Install Cilium:**

   ```bash
   helm install cilium cilium/cilium \
     --namespace kube-system \
     --version 1.19.2 \
     --set hubble.enabled=true \
     --set hubble.relay.enabled=true \
     --set hubble.metrics.enabled="{dns,drop,tcp,flow,icmp,http}"
   ```

1. **Wait for Cilium to be ready:**

   ```bash
   kubectl -n kube-system rollout status daemonset/cilium --timeout=300s
   kubectl -n kube-system rollout status deployment/cilium-operator --timeout=120s
   kubectl -n kube-system rollout status deployment/hubble-relay --timeout=120s
   ```

1. **Restart all pods** to re-acquire network identities under Cilium:

   ```bash
   kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name \
     --no-headers | grep -v kube-system | \
     while read ns name; do kubectl delete pod "$name" -n "$ns" --grace-period=30; done
   ```

   Or, for a less disruptive approach, restart namespace by namespace.

1. **Verify networking:**

   ```bash
   # Check Cilium status
   kubectl -n kube-system exec ds/cilium -- cilium status

   # Check all pods are running
   kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded

   # Test cross-pod connectivity
   kubectl run test-a --image=busybox --restart=Never -- sleep 3600
   kubectl run test-b --image=busybox --restart=Never -- sleep 3600
   kubectl exec test-a -- ping -c 3 $(kubectl get pod test-b -o jsonpath='{.status.podIP}')
   kubectl delete pod test-a test-b
   ```

### Post-Migration (This Repo)

1. ArgoCD syncs the `networkanalyzer` app, deploying Hubble UI into `network-observability`.
1. Access Hubble UI:

   ```bash
   kubectl port-forward -n network-observability svc/hubble-ui 12000:80
   ```

   Open `http://localhost:12000`.

1. Verify Hubble UI shows flow data from the cluster.

## Rollback Plan

If Cilium installation fails or causes persistent network issues:

### Quick Rollback (Restore Calico)

1. Remove Cilium:

   ```bash
   helm uninstall cilium -n kube-system
   ```

1. Re-install Calico using the method it was originally deployed with.

1. Restart all pods to re-acquire Calico networking.

1. Verify cluster networking is restored.

### Hubble UI Impact

If Cilium is rolled back:

- Hubble UI will deploy but show no data (Relay won't be available)
- No impact on the infra-telemetry or flow-analytics planes
- The Hubble UI Deployment will enter a reconnection loop — harmless but not functional

## Calico NetworkPolicy Migration

If the cluster has existing Calico NetworkPolicies:

- **Standard Kubernetes NetworkPolicies** are compatible with Cilium — no changes needed.
- **Calico-specific policies** (GlobalNetworkPolicy, NetworkSet, etc.) must be migrated to CiliumNetworkPolicy or CiliumClusterwideNetworkPolicy equivalents.
- Audit existing policies before migration:

  ```bash
  kubectl get networkpolicies --all-namespaces
  kubectl get globalnetworkpolicies 2>/dev/null || echo "No Calico GlobalNetworkPolicies"
  ```

## Enabling L7 Visibility

After Cilium is installed, L7 visibility (HTTP methods/paths, gRPC methods) requires one of:

### Option 1: Pod Annotation (Per-Workload)

```yaml
metadata:
  annotations:
    policy.cilium.io/proxy-visibility: "<Ingress/80/TCP/HTTP>"
```

### Option 2: CiliumNetworkPolicy with L7 Rules

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-http-visibility
spec:
  endpointSelector:
    matchLabels:
      app: my-app
  ingress:
    - toPorts:
        - ports:
            - port: "80"
              protocol: TCP
          rules:
            http:
              - method: "GET"
```

Without either of these, Hubble shows L3/L4 flow data only (IPs, ports, bytes) — which is still useful for workload/service mapping.
