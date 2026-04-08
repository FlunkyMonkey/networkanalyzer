# GitOps Structure

## Directory Layout

```text
platform/
├── apps/
│   └── network-observability.yaml   # Single ArgoCD Application
├── base/
│   ├── kustomization.yaml           # Root kustomization composing all planes
│   ├── namespace.yaml               # network-observability namespace
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
- The overlay references `platform/base`, which composes all four planes.
- ArgoCD sees a single sync unit — all planes deploy together as one logical application.

This avoids app-of-apps complexity while keeping plane manifests separated for readability and ownership.

## How ArgoCD Composes the Platform

```text
ArgoCD Application
  └── platform/overlays/lab/kustomization.yaml
        └── ../../base/kustomization.yaml
              ├── namespace.yaml
              ├── infra-telemetry/kustomization.yaml
              ├── flow-analytics/kustomization.yaml
              ├── k8s-visibility/kustomization.yaml
              └── correlation-ux/kustomization.yaml
```

ArgoCD runs `kustomize build` on the overlay path. The overlay inherits everything from base and can add patches, resource limits, or environment-specific configuration.

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

2. Reference the base:

   ```yaml
   resources:
     - ../../base
   ```

3. Add environment-specific patches as needed.

4. Update the ArgoCD Application (or create a copy) to point at the new overlay path.
