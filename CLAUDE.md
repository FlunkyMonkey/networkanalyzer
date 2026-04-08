# Network Observability Platform — Project Instructions

## Purpose

This repository defines a GitOps-managed network observability platform deployed to Kubernetes via ArgoCD.

The platform must provide:

1. Bandwidth utilization by interfaces, including:
   - switch interfaces
   - WLAN clients
   - Proxmox VMs
2. Traffic analysis including:
   - top talkers
   - destinations / websites
   - apps
   - categories
   - countries
   - ports
3. Robust but elegant visualization
4. Deep Kubernetes network visibility with L7-aware drill-down from the start
5. A polished primary UI with specialist drill-downs
6. Health, freshness, and degraded-state visibility across all telemetry planes

## Non-Negotiable Architecture Principles

1. Use separate telemetry planes with a unified user experience.
2. Prefer the cleaner long-term design over short-term shortcuts.
3. Keep raw source truth in each telemetry plane.
4. Centralize navigation and entity correlation in the platform experience.
5. No silent data loss or silent stale-data states.
6. All major services must expose health and readiness.
7. All implementations must be GitOps-friendly and ArgoCD-deployable.
8. All stateful services deployed in Kubernetes must use PVC-backed storage.
9. Shared Ceph storage is the default unless explicitly changed.
10. Do not invent broad refactors or architecture changes without explicit approval.

## Current Architectural Direction

The platform is expected to contain these logical planes:

- Infrastructure / telemetry plane
- Flow analytics plane
- Kubernetes network visibility plane
- Correlation / UX plane

Do not collapse these into a single product unless explicitly directed.

## User Experience Priorities

Homepage card order is locked:

1. Top talkers
2. What websites / destinations server X is connecting to
3. Bandwidth utilization of WiFi Client 7

Default investigation flow is locked:

WAN -> work downward

The platform must support:
- fast loading
- entity search
- cross-plane drill-downs
- path-style relationship visualization
- degraded-state banners
- saved investigation playbooks

## Canonical Entity Model

Design and naming should align to these entities:

### Network Asset
- wan
- firewall
- router
- vlan
- switch
- switch_interface
- ap
- ssid

### Client / Endpoint
- ip
- mac
- hostname
- dns_name
- country
- asn
- app
- category

### Compute Asset
- proxmox_host
- vm
- k8s_node

### Kubernetes Workload
- cluster
- namespace
- workload
- pod
- service

### Flow
- src
- dst
- protocol
- port
- bytes
- packets
- timestamps
- geo
- app
- category

### Incident Context
- alert
- severity
- preserved_window
- linked_entities

Use these names consistently in docs, code, dashboards, and manifests.

## Development Rules

When implementing:

1. Inspect existing repo patterns before creating new ones.
2. Make the smallest reasonable change for the requested task.
3. Do not change unrelated files.
4. Do not add secrets, fake credentials, or inline private values.
5. Explain every new dependency before adding it.
6. Prefer explicit and readable configuration over cleverness.
7. Keep changes easy to review.
8. Update docs when behavior or structure changes.
9. Return evidence, not just conclusions.

## Required Output For Each Task

For every meaningful task, return:

- short plan
- summary of changes
- files changed
- commands run
- results
- open risks
- next recommended step

Do not respond with vague statements like "done" without evidence.

## Validation Expectations

At minimum, validate relevant changes using the repo’s available tooling.

Examples:
- helm lint
- helm template
- kustomize build
- yaml validation
- json schema validation
- manifest rendering
- linting scripts

If validation fails:
- do not hide it
- do not pretend the task is complete
- explain the failure clearly

## ArgoCD / Kubernetes Conventions

Unless the repo already defines a different convention:

- keep environments and overlays structured clearly
- separate app concerns cleanly
- define namespaces intentionally
- use readiness and liveness checks where appropriate
- define storage requirements explicitly for stateful services
- avoid hidden coupling between components

## Observability Requirements For The Platform Itself

The observability platform must observe itself.

Implement or plan for:
- source freshness
- ingest lag
- dropped data indicators
- stale-data banners
- backend health
- degraded-state visibility
- operator troubleshooting views

## Testing and Acceptance

Initial promotion gate:
- lint required

Acceptance evidence should eventually cover:
- interface utilization
- WiFi client utilization
- VM visibility
- top talkers
- destination analysis
- app/category/country/port pivots
- Kubernetes workload visibility
- path-style correlation
- degraded-state banner behavior
- alert-window preservation

## What Requires Explicit Approval

Do not do any of the following without explicit approval:

- major architecture changes
- introducing a new core platform dependency
- changing storage class or persistence strategy broadly
- restructuring the repo significantly
- replacing one telemetry plane with another
- changing the canonical entity model
- introducing secrets management tooling not already approved

## Preferred Working Style

1. Return a short plan before editing.
2. Make minimal, focused changes.
3. Validate them.
4. Summarize exactly what changed and why.
5. Call out risks and ambiguities clearly.

## Definition of Done

A task is done when:
- the requested scoped change is implemented
- related validation has been run
- results are reported clearly
- docs are updated if needed
- no unrelated drift was introduced
