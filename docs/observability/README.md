# Network Observability Platform

This directory contains design documentation for the GitOps-managed network observability platform.

## Contents

| Document | Description |
|---|---|
| [architecture.md](architecture.md) | Logical planes, canonical entity model, UX priorities, retention model |
| [phases.md](phases.md) | Phase 0 through Phase 9 roadmap with deliverables |
| [acceptance-tests.md](acceptance-tests.md) | Acceptance test outline by domain |
| [source-mapping.md](source-mapping.md) | Source-to-signal matrix and requirements mapping |
| [gitops-structure.md](gitops-structure.md) | GitOps layout, ArgoCD model, and contribution guide |
| [telemetry-plane.md](telemetry-plane.md) | Infrastructure telemetry components, metrics, and integration details |
| [secrets-and-bootstrap.md](secrets-and-bootstrap.md) | Required secrets, bootstrap steps, and network prerequisites |
| [flow-analytics-plane.md](flow-analytics-plane.md) | Flow analytics components, enrichment, and investigation capabilities |
| [flow-ingest-and-cutover.md](flow-ingest-and-cutover.md) | Softflowd cutover steps and OpenSearch bootstrap procedures |
| [k8s-visibility-plane.md](k8s-visibility-plane.md) | Kubernetes visibility architecture, ownership boundary, and capabilities |
| [cni-migration-and-rollout.md](cni-migration-and-rollout.md) | Calico → Cilium migration runbook (prerequisite, managed in k8s-lab.git) |
| [unified-ux-and-correlation.md](unified-ux-and-correlation.md) | Unified UX approach, dashboard organization, cross-plane data access |
| [navigation-model.md](navigation-model.md) | Navigation structure, drill-down paths, link implementation |
| [investigation-playbooks.md](investigation-playbooks.md) | Guided investigation workflows for common network scenarios |
| [deployment-plan.md](deployment-plan.md) | Phased rollout waves (Wave 0–5) with execution order and rollback |
| [rollout-checklists.md](rollout-checklists.md) | Operator-executable checklists for each rollout wave |
| [operator-test-procedures.md](operator-test-procedures.md) | Detailed test cases (T1–T11) for operator execution |
| [regression-test-plan.md](regression-test-plan.md) | Regression matrix to run after each rollout wave |
| [codex-qa-sow.md](codex-qa-sow.md) | Independent QA/regression subcontract scope for Codex |
| [go-no-go-gates.md](go-no-go-gates.md) | Explicit approval gates for each rollout wave |
| [wave-enablement-model.md](wave-enablement-model.md) | How phased rollout works with a single ArgoCD app |
| [gemini-strategy-sow.md](gemini-strategy-sow.md) | External strategic review role for GeminiCLI |
| [wave2-canary-cutover.md](wave2-canary-cutover.md) | Wave 3 canary cutover runbook: prox5 first, rollback to 172.18.1.207:2055 |
| [wave2-codex-qa.md](wave2-codex-qa.md) | Codex QA packet for Wave 2/3 readiness |
| [wave-2-closeout.md](wave-2-closeout.md) | Wave 2 formal closeout: acceptance state, residual conditions, sign-off |
| [wave-3-plan.md](wave-3-plan.md) | Wave 3 plan: Kubernetes flow correlation, enrichment strategy, phased implementation |
| [wave-3b-closeout.md](wave-3b-closeout.md) | Wave 3b formal closeout: K8s IP correlation accepted, hotfixes, caveats, runbook |
| [wave-4-plan.md](wave-4-plan.md) | Wave 4 plan: Cilium/Hubble K8s visibility — gated on CNI migration, Phase 1 (discovery) active |
| [backlog.md](backlog.md) | Hardening items and future-wave work not blocking active rollout |
| [operations-notes.md](operations-notes.md) | Runtime ops decisions and context (softflowd service pattern, legacy cleanup) |

## Platform Structure

The platform manifests live under `platform/` in the repo root:

```text
platform/
├── apps/                  # ArgoCD Application manifest (single-app model)
├── base/                  # Base manifests per telemetry plane
│   ├── infra-telemetry/   # SNMP, syslog, device polling
│   ├── flow-analytics/    # NetFlow/sFlow, enrichment, analytics
│   ├── k8s-visibility/    # Hubble UI (standalone); Cilium managed in k8s-lab.git
│   └── correlation-ux/    # Unified UI, entity correlation, search
└── overlays/              # Environment-specific patches
```

## Deployment Model

- GitOps via ArgoCD
- Single ArgoCD Application managing the platform namespace
- Kustomize overlays for environment differentiation
- PVC-backed storage for all stateful services (Ceph default)
