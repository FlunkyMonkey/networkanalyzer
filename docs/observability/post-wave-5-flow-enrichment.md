# Post-Wave 5 Flow Enrichment Plan

## Status: Phase 1 — Design and Audit Complete

Phase 1 audit completed 2026-05-02. Design is ready for Phase 2 implementation.

---

## Background

Wave 5 is complete. The flow pipeline ingests ~3.7M flows/day and enriches them
with K8s workload context (Wave 3b). Two remaining visibility gaps:

1. **Unlabeled high-volume traffic** — many of the top flows by byte volume have
   no `app` label. The Traffic Mix "Traffic by App Label" panel shows only the
   flows on well-known ports; most Ceph storage and K8s control-plane traffic is
   invisible.
2. **Raw IPs in dashboards** — the Top Talkers and Destination Analysis tables
   show `172.18.1.20`, `172.18.1.96`, etc. Operators must know the homelab IP
   map to interpret them. Friendly names are needed.

---

## Phase 1 Audit Findings

### Top Unlabeled Destination Ports (2026-05-02, today's index)

| Rank | Port  | Volume   | Flows   | Identification                        |
|------|-------|----------|---------|---------------------------------------|
| 1    | 6802  | 20.85 GiB|      19 | **Ceph OSD msgr2** (rook-ceph-osd-*)  |
| 2    | 6801  | 19.81 GiB|     686 | **Ceph OSD msgr2** (rook-ceph-osd-*)  |
| 3    | 786   |  9.98 GiB|     175 | **NFS mountd** (Proxmox ↔ TrueNAS)   |
| 4    | 44729 |  8.59 GiB|      13 | Ephemeral — kubelet log streaming      |
| 5    | 813   |  8.01 GiB|     136 | **NFS aux** (TrueNAS/Proxmox)         |
| 6    | 42020 |  6.44 GiB|      69 | Ephemeral — kube-state-metrics scrape  |
| 7    | 747   |  5.74 GiB|     113 | **NFS aux** (TrueNAS/Proxmox)         |
| 8    | 3300  |  5.19 GiB|  86,415 | **Ceph monitor msgr2** (rook-ceph-mon) |
| 9    | 2380  |  4.54 GiB| 237,342 | **etcd peer** (k8s control plane)      |
| 10–17| varies|  ~4.3 GiB each | 50–90 | Ephemeral — Ceph OSD replication     |
| 18   | 741   |  3.81 GiB|     128 | **NFS aux** (TrueNAS/Proxmox)         |
| 19   | 51260 |  3.55 GiB|      73 | Ephemeral — k8s node traffic           |
| 20   | 6808  |  2.44 GiB|       7 | **Ceph MGR msgr2**                    |

**Key finding:** The top unlabeled traffic splits into three groups:

1. **Static identifiable ports** (6801, 6802, 6808, 3300, 786, 813, 747, 741,
   2380) — can be labeled by adding to the port registry.
2. **K8s workload-identifiable flows** (Ceph OSD replication, Prometheus scrapes)
   — these use ephemeral destination ports that are dynamically assigned. They
   already have `dst_k8s_workload` set via Wave 3b enrichment. Should be labeled
   by workload pattern, not by port number.
3. **Kubelet ephemeral ports** (44729, etc.) — vary per session. Cannot be labeled
   by port. Already identified as `src_k8s_type: node`.

### Currently Labeled Traffic (for context)

| App              | Volume today |
|------------------|-------------|
| NFS              | 149.00 GiB  |
| OpenSearch       |   2.51 GiB  |
| Kubernetes (6443)|   0.32 GiB  |
| HTTP-Alt (8080)  |   0.18 GiB  |
| Proxmox (8006)   |   0.10 GiB  |
| (rest)           |  <0.1 GiB   |

NFS traffic (port 2049) is already labeled — it accounts for 149 GiB destined to
`truenas01` (172.18.1.96). Port 2049 is already in the Vector `port_apps` map.
The unlabeled 786/813/747/741 are the NFS protocol *negotiation* ports (mountd,
statd, lockd) — separate from the data-plane port 2049.

### Top Source/Destination IPs (2026-05-02)

| IP            | Volume   | Current label     | Friendly name        |
|---------------|----------|-------------------|----------------------|
| 172.18.1.20   | 51.6 GiB | external          | prox2.vgriz.com      |
| 172.18.1.30   | 42.0 GiB | external          | prox3.vgriz.com      |
| 172.18.1.40   | 38.7 GiB | external          | prox4.vgriz.com      |
| 172.18.1.96   | 27.6 GiB | external          | truenas01.vgriz.com  |
| 172.18.1.62   | 24.4 GiB | node / kube2      | kube2.vgriz.com      |
| 172.18.1.64   | 17.0 GiB | node / kube4      | kube4.vgriz.com      |
| 172.18.1.10   | 16.9 GiB | external          | prox1.vgriz.com      |
| 172.18.1.61   | 15.8 GiB | node / kube1      | kube1.vgriz.com      |
| 172.18.1.63   |  8.4 GiB | node / kube3      | kube3.vgriz.com      |
| 172.18.1.15   |  0.6 GiB | external          | unifi.vgriz.com      |
| 172.18.232.10 |  low     | external          | dns1.vgriz.com       |

**Key finding:** The top 4 `external` IPs by volume are Proxmox hosts and
TrueNAS01. These are high-value enrichment targets. K8s nodes already have
`src_k8s_node` set to the node name — they benefit less from hostname enrichment
since their context is already visible.

---

## Design

### A. App-to-Port Registry

**Format:** CSV file at `config/enrichment/app-port-registry.csv`

**Fields:**

```text
port, protocol, app, category, direction, scope, notes
```

- `port` — integer destination port
- `protocol` — TCP, UDP, or `*` for both
- `app` — short label used in the `app` field (same namespace as existing labels)
- `category` — storage, control-plane, monitoring, database, etc.
- `direction` — `dst` (label when matched as dst_port), `src` (src_port), `both`
- `scope` — optional: `internal` (RFC1918 only), `external`, or blank (any)
- `notes` — free text, not used by Vector

**Why CSV:** Vector supports `file` enrichment tables with CSV encoding natively —
the same mechanism as the k8s-lookup-tables ConfigMap. No preprocessing or build
step is required. The file can be mounted from a ConfigMap and consumed directly
by Vector's `get_enrichment_table_record` VRL function.

**Sample file:** `config/enrichment/app-port-registry.csv.sample`

### B. Workload-Based App Labeling (VRL extension)

For Ceph OSD replication flows, the ephemeral dst_port cannot identify them, but
the K8s workload name can. A short VRL block in `enrich_meta` checks `dst_k8s_workload`
after the port lookup and overwrites `.app` when a workload-pattern match fires:

```vrl
# Workload-pattern labeling for ephemeral-port traffic
if is_null(.app) {
  wl = to_string(.dst_k8s_workload) ?? ""
  if starts_with(wl, "rook-ceph-osd") {
    .app = "Ceph-OSD"
  } else if starts_with(wl, "rook-ceph-mon") {
    .app = "Ceph-Monitor"
  } else if starts_with(wl, "rook-ceph-mgr") {
    .app = "Ceph-Manager"
  } else if starts_with(wl, "prometheus-") || starts_with(wl, "kube-prometheus") {
    .app = "Prometheus-Internal"
  }
}
```

This is purely in VRL — no lookup table required. It runs after the port lookup so
the port registry takes precedence over workload patterns.

This is proposed for Phase 3 (Vector config change), not Phase 1.

### C. Hostname / Friendly-Name Enrichment

**Recommended approach: Static repo-managed host map (CSV lookup table)**

**Rationale for static over runtime CronJob:**

The homelab has approximately 20 stable infrastructure hosts in `172.18.1.x` and
related subnets. These rarely change. The full DNS zone is already documented in
`/Users/claude/code/homelab/homelab-dns-zones.md`. A static CSV:

- Is trivially reviewable in Git (plain IP → name mapping)
- Requires no CronJob, no DNS API access, no credentials
- Has zero runtime failure modes
- Stays in sync when operators update it alongside DNS changes

A runtime CronJob querying BIND DNS would be appropriate if the fleet is large or
dynamic. For a stable 20-host homelab, it adds complexity with no benefit.

**Format:** CSV file at `config/enrichment/hostname-map.csv`

```text
ip, hostname, short_name, device_type, notes
```

- `ip` — IPv4 address (exact match)
- `hostname` — FQDN from BIND zone (e.g., `prox2.vgriz.com`)
- `short_name` — operator-friendly short label (e.g., `prox2`)
- `device_type` — proxmox, truenas, kube-node, switch, gateway, dns, ap
- `notes` — free text

**Vector enrichment:** adds `src_hostname` and `dst_hostname` fields from a lookup
table. Falls back to the raw IP if no match.

**Dashboard display format:**

- Known host: `prox2 (172.18.1.20)`
- K8s node: `kube2 (172.18.1.62)` — already available via `src_k8s_node`
- RFC1918, no match: `unknown-172.18.1.x`
- Public IP: raw IP (GeoIP enrichment will add country/ASN in Wave 7)

**Sample file:** `config/enrichment/hostname-map.csv.sample`

**Note on K8s nodes:** K8s node hostnames are already available via the Wave 3b
`src_k8s_node` / `dst_k8s_node` fields. The hostname map entries for kube nodes
are provided for completeness but dashboards should prefer the k8s enrichment field
over a lookup-table result for flows classified as `k8s_type: node`.

---

## Implementation Sequence

### Phase 2 — Registry and Lookup Table Plumbing

**Scope:** Wire both CSV files into Vector and add the `src_hostname` / `dst_hostname`
enrichment fields. No dashboard changes yet.

1. Promote `app-port-registry.csv.sample` → `app-port-registry.csv` (finalize
   port entries).
2. Promote `hostname-map.csv.sample` → `hostname-map.csv` (finalize host entries).
3. Add both files to the `k8s-lookup-tables` ConfigMap (or a new
   `flow-enrichment-tables` ConfigMap) so Vector can mount them.
4. Add two new `enrichment_tables` entries to `vector-config.yaml`.
5. Extend `enrich_meta` VRL transform to:
   - Look up `dst_port` in `app_ports` table and set `.app` (replaces the
     current hardcoded `port_apps` object in VRL).
   - Look up `src_ip` and `dst_ip` in `hostname_map` table and set
     `src_hostname` / `dst_hostname`.
   - Apply `unknown-{ip}` fallback for RFC1918 addresses with no match.
6. Validate: deploy to cluster, confirm new fields appear in flow documents.

**Decision required before Phase 2:** Should the port registry and hostname map be
in the same `k8s-lookup-tables` ConfigMap (one ConfigMap, multiple CSV keys) or a
new separate `flow-enrichment-tables` ConfigMap? Recommendation: separate ConfigMap
so it has independent `ignoreDifferences` control and doesn't couple with the
CronJob-managed k8s tables.

### Phase 3 — Vector Enrichment Fields

**Scope:** Add workload-based app labeling VRL, validate, deploy.

1. Add the workload-pattern VRL block to `enrich_meta` (see section B above).
2. Add `app_category` field derived from the registry `category` column.
3. Update the OpenSearch index template to add `src_hostname`, `dst_hostname`,
   and `app_category` as keyword fields.
4. Validate: confirm Ceph OSD flows show `app: Ceph-OSD` in new documents.

### Phase 4 — Dashboard Label Updates

**Scope:** Surface `src_hostname`, `dst_hostname`, and `app_category` in dashboards.

1. **Top Talkers:** rename "Source IP" column to "Source" and display
   `src_hostname ?? src_ip` using a Grafana value mapping or transform.
   Format: `short_name (ip)` where available, raw IP otherwise.
2. **Destination Analysis:** same treatment for `dst_ip` / `dst_hostname`.
3. **Traffic Mix:** add `app_category` piechart alongside the existing `app` chart.
4. **K8s Context:** already shows namespace/workload — hostname enrichment is
   additive here, lower priority.
5. Update `Sum bytes` → `Total Volume` field labels in any panels not yet renamed.

### Phase 5 — Validation and Closeout

1. Verify top unlabeled ports are now labeled in Traffic Mix.
2. Verify Proxmox/TrueNAS hosts show friendly names in Top Talkers.
3. Verify no regression in existing K8s enrichment fields.
4. Update backlog to mark post-wave-5 items complete.
5. Run `kustomize build` and markdownlint.
6. Commit closeout.

---

## Open Questions (require decision before Phase 2)

1. **ConfigMap placement:** new `flow-enrichment-tables` ConfigMap vs. extending
   `k8s-lookup-tables`? Recommendation: new ConfigMap (cleaner separation).

2. **Port registry source of truth:** Keep the static VRL `port_apps` object AND
   add a lookup table, OR fully replace the object with the lookup table?
   Recommendation: replace the VRL object with the lookup table — single source of
   truth, easier to maintain, same Vector mechanism.

3. **Hostname field name:** `src_hostname`/`dst_hostname` or `src_name`/`dst_name`?
   Recommendation: `src_hostname`/`dst_hostname` — unambiguous and consistent with
   DNS terminology.

4. **Dashboard display for K8s nodes:** nodes already have `src_k8s_node` set to
   the node name (e.g., `kube2`). Should dashboard hostname lookup be skipped when
   `src_k8s_type == "node"` (using `src_k8s_node` instead) to avoid duplicating
   the enrichment? Recommendation: yes — prefer k8s enrichment fields over
   hostname lookup for node-type flows.

---

## Files in This Commit

| File | Purpose |
|------|---------|
| `docs/observability/post-wave-5-flow-enrichment.md` | This design doc |
| `config/enrichment/app-port-registry.csv.sample` | Port registry scaffold with audit findings |
| `config/enrichment/hostname-map.csv.sample` | Hostname map scaffold from DNS zone |
| `config/enrichment/README.md` | Brief directory README |

No manifests, Vector config, CronJobs, or dashboards are changed in this commit.
