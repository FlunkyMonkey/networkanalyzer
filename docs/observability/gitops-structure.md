# GitOps Structure

## Directory Layout

```text
platform/
├── apps/
│   └── network-observability.yaml   # Single ArgoCD Application
├── base/
│   ├── kustomization.yaml           # Root kustomization composing all planes
│   ├── common/
│   │   ├── kustomization.yaml       # Shared resources (namespace)
│   │   └── namespace.yaml           # network-observability namespace
│   ├── infra-telemetry/
│   │   └── kustomization.yaml       # Infrastructure / Telemetry plane
│   ├── flow-analytics/
│   │   └── kustomization.yaml       # Flow Analytics plane
│   ├── k8s-visibility/
│   │   └── kustomization.yaml       # Kubernetes Network Visibility plane
│   └── correlation-ux/
│       └── kustomization.yaml       # Correlation / UX plane
└── overlays/
    └── lab/
        └── kustomization.yaml       # Lab environment overlay
```

## Single ArgoCD Application Model

The platform is managed by **one** ArgoCD Application (`platform/apps/network-observability.yaml`).

- The Application points at an overlay (e.g., `platform/overlays/lab`).
- The overlay selectively includes planes from `platform/base/` — not all at once.
- Planes are enabled incrementally by uncommenting resource lines in the overlay (wave-gated rollout).
- ArgoCD sees a single sync unit — enabled planes deploy together as one logical application.

This avoids app-of-apps complexity while supporting phased rollout. See [wave-enablement-model.md](wave-enablement-model.md) for the full model.

## How ArgoCD Composes the Platform

```text
ArgoCD Application
  └── platform/overlays/lab/kustomization.yaml
        ├── ../../base/common                    (always)
        ├── ../../base/infra-telemetry           (Wave 1 — enabled)
        ├── ../../base/flow-analytics            (Wave 2 — uncomment to enable)
        ├── ../../base/k8s-visibility            (Wave 4 — uncomment to enable)
        └── ../../base/correlation-ux            (Wave 5 — uncomment to enable)
```

ArgoCD runs `kustomize build` on the overlay path. The overlay selectively includes planes and can add patches, resource limits, or environment-specific configuration. `platform/base/kustomization.yaml` still composes all planes as a reusable building block for future environments.

## Adding Resources to a Plane

To add a new manifest to a plane (e.g., an SNMP collector to infra-telemetry):

1. Create the manifest file in the plane directory:

   ```text
   platform/base/infra-telemetry/snmp-collector-deployment.yaml
   ```

2. Add it to the plane's `kustomization.yaml`:

   ```yaml
   resources:
     - snmp-collector-deployment.yaml
   ```

3. Commit and push. ArgoCD will sync the new resource automatically.

No changes to the ArgoCD Application manifest or root kustomization are needed.

## What Belongs in Base vs. Overlays

### Base (`platform/base/`)

- Namespace definition
- All plane kustomizations and their manifests
- Common labels (`app.kubernetes.io/part-of: network-observability`)
- Resource definitions that apply to all environments

### Overlays (`platform/overlays/<env>/`)

- Environment-specific patches (resource limits, replica counts)
- Storage class overrides
- Environment-specific ConfigMaps or annotations
- Any configuration that differs between environments

## Common Labels

All resources inherit these labels via the root kustomization:

| Label | Value |
|---|---|
| `app.kubernetes.io/part-of` | `network-observability` |

Each plane adds its own component label:

| Plane | `app.kubernetes.io/component` |
|---|---|
| Infrastructure / Telemetry | `infra-telemetry` |
| Flow Analytics | `flow-analytics` |
| Kubernetes Visibility | `k8s-visibility` |
| Correlation / UX | `correlation-ux` |

## Adding a New Environment Overlay

1. Create the overlay directory:

   ```text
   platform/overlays/staging/kustomization.yaml
   ```

2. Reference the planes you want (or the full base for an all-at-once deployment):

   ```yaml
   # Wave-gated (selective):
   resources:
     - ../../base/common
     - ../../base/infra-telemetry
     # - ../../base/flow-analytics    # enable when ready

   # Or all-at-once:
   resources:
     - ../../base
   ```

3. Add environment-specific patches as needed.

4. Update the ArgoCD Application (or create a copy) to point at the new overlay path.
