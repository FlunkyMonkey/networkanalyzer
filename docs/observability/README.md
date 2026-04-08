# Network Observability Platform

This directory contains design documentation for the GitOps-managed network observability platform.

## Contents

| Document | Description |
|---|---|
| [architecture.md](architecture.md) | Logical planes, canonical entity model, UX priorities, retention model |
| [phases.md](phases.md) | Phase 0 through Phase 8 roadmap with deliverables |
| [acceptance-tests.md](acceptance-tests.md) | Acceptance test outline by domain |
| [source-mapping.md](source-mapping.md) | Source-to-signal matrix and requirements mapping |
| [gitops-structure.md](gitops-structure.md) | GitOps layout, ArgoCD model, and contribution guide |
| [telemetry-plane.md](telemetry-plane.md) | Infrastructure telemetry components, metrics, and integration details |
| [secrets-and-bootstrap.md](secrets-and-bootstrap.md) | Required secrets, bootstrap steps, and network prerequisites |

## Platform Structure

The platform manifests live under `platform/` in the repo root:

```text
platform/
├── apps/                  # ArgoCD Application manifest (single-app model)
├── base/                  # Base manifests per telemetry plane
│   ├── infra-telemetry/   # SNMP, syslog, device polling
│   ├── flow-analytics/    # NetFlow/sFlow, enrichment, analytics
│   ├── k8s-visibility/    # eBPF, service mesh, L7 visibility
│   └── correlation-ux/    # Unified UI, entity correlation, search
└── overlays/              # Environment-specific patches
```

## Deployment Model

- GitOps via ArgoCD
- Single ArgoCD Application managing the platform namespace
- Kustomize overlays for environment differentiation
- PVC-backed storage for all stateful services (Ceph default)
