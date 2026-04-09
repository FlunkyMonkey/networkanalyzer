# Wave Enablement Model

## How Phased Rollout Works with a Single ArgoCD App

The platform uses one ArgoCD Application pointing at `platform/overlays/lab/`. Phased rollout is achieved by controlling which planes are included in the lab overlay's `kustomization.yaml`.

Each wave is enabled by a small repo commit that uncomments the next plane's resource line. ArgoCD detects the change and syncs the new resources.

```text
platform/apps/network-observability.yaml
  → points at: platform/overlays/lab/
    → lab kustomization.yaml selectively includes planes:
        ├── ../../base/common               (always — namespace)
        ├── ../../base/infra-telemetry       (Wave 1)
        ├── ../../base/flow-analytics        (Wave 2 — uncomment to enable)
        ├── ../../base/k8s-visibility        (Wave 4 — uncomment to enable)
        └── ../../base/correlation-ux        (Wave 5 — uncomment to enable)
```

## Wave Activation Sequence

| Wave | Action | Repo Change | ArgoCD Effect |
|---|---|---|---|
| 0 | Preflight | None | App not yet applied |
| 1 | Apply ArgoCD app | `kubectl apply -f platform/apps/network-observability.yaml` | Deploys namespace + infra-telemetry |
| 2 | Enable flow-analytics | Uncomment `../../base/flow-analytics` in lab overlay, commit, push | ArgoCD syncs: adds OpenSearch, flow collector |
| 3 | Softflowd cutover | Operational — repoint softflowd on Proxmox hosts | No repo change needed |
| 4 | Enable k8s-visibility | Uncomment `../../base/k8s-visibility` in lab overlay, commit, push | ArgoCD syncs: adds Hubble UI |
| 5 | Enable correlation-ux | Uncomment `../../base/correlation-ux` in lab overlay, commit, push | ArgoCD syncs: adds dashboards, datasource |

## How to Enable a Wave

1. Open `platform/overlays/lab/kustomization.yaml`
2. Uncomment the resource line for the next plane
3. Commit and push to `main`
4. ArgoCD auto-syncs (or manual sync: `argocd app sync network-observability`)
5. Verify new pods are healthy
6. Run the wave's checklist and test procedures
7. Pass the wave's go/no-go gate before enabling the next wave

### Example: Enabling Wave 2

```yaml
# Before (Wave 1 only):
resources:
  - ../../base/common
  - ../../base/infra-telemetry
  # - ../../base/flow-analytics       # <-- commented out

# After (Wave 2 enabled):
resources:
  - ../../base/common
  - ../../base/infra-telemetry
  - ../../base/flow-analytics          # <-- uncommented
```

Commit message: `Wave 2: enable flow-analytics plane`

## Rollback

To roll back a wave, re-comment the resource line, commit, and push. ArgoCD will prune the removed resources (automated prune is enabled in the Application spec).

## Why Not Use the Full Base Kustomization?

`platform/base/kustomization.yaml` composes all planes and is kept as a reusable building block. The lab overlay selectively includes individual planes instead of referencing the full base, so:

- Each wave is an explicit, reviewable repo change
- The operator controls rollout pace
- Rollback is a one-line change
- The full base remains available for future environments that want everything at once

## Relationship to Other Docs

- [deployment-plan.md](deployment-plan.md) — wave purposes, prerequisites, execution order
- [rollout-checklists.md](rollout-checklists.md) — operator checklists per wave
- [go-no-go-gates.md](go-no-go-gates.md) — approval gates between waves
- [gitops-structure.md](gitops-structure.md) — overall repo layout
