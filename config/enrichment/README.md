# config/enrichment

Enrichment lookup tables for the flow analytics pipeline.

These are data files (not manifests). They are used in Phase 2 of Post-Wave 5
Flow Enrichment to extend Vector's enrichment capabilities beyond the existing
K8s IP lookup tables.

## Files

| File | Status | Purpose |
|------|--------|---------|
| `app-port-registry.csv.sample` | Scaffold — not yet wired | Port-to-app mapping registry |
| `hostname-map.csv.sample` | Scaffold — not yet wired | IP-to-hostname friendly-name map |

## Lifecycle

1. Review `.sample` files after Phase 1 audit.
2. Rename to `.csv` (remove `.sample` suffix) when ready to wire.
3. Phase 2 adds these to a `flow-enrichment-tables` ConfigMap and references
   them as Vector `enrichment_tables` in `vector-config.yaml`.
4. Phase 3 extends VRL to set `app_category`, workload-based `app` labels,
   `src_hostname`, and `dst_hostname`.

## No secrets

These files contain only IP addresses, port numbers, hostnames, and labels.
No credentials, API keys, or passwords. All data is either public DNS zone
information or well-known port assignments.

## Updating

- **Add a new host:** append a row to `hostname-map.csv`.
- **Add a new port mapping:** append a row to `app-port-registry.csv`.
- Both changes take effect after ArgoCD syncs the `flow-enrichment-tables`
  ConfigMap and `flow-collector` is restarted (Vector does not live-reload
  enrichment tables — same limitation as Wave 3b k8s-lookup-tables).
