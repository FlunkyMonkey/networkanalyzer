# Post-Wave 5 Flow Enrichment Plan

## Status: Phase 5 — Validated and Closed Out

Phase 1 audit completed 2026-05-02. Phase 2 wiring deployed 2026-05-02.
Phase 3a mapping deployed 2026-05-03. Phase 3b workload labeling deployed 2026-05-03.
Phase 4 dashboard updates deployed 2026-05-03. Phase 5 validated and closed out
2026-06-08 on live data (`flows-2026.06.08`, 3.48M docs).

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

### Phase 2 — Registry and Lookup Table Plumbing — COMPLETE (2026-05-02)

**Decision:** New `flow-enrichment-tables` ConfigMap (separate from `k8s-lookup-tables`).
Reason: `k8s-lookup-tables` is runtime-patched by the CronJob/PostSync. Enrichment tables
are GitOps-owned operator-curated data with a different lifecycle.

**What was implemented:**

1. Promoted `app-port-registry.csv.sample` → `config/enrichment/app-port-registry.csv`
   (53 entries; duplicate 2049/TCP entry removed; comment lines stripped).
2. Promoted `hostname-map.csv.sample` → `config/enrichment/hostname-map.csv`
   (36 entries; comment lines stripped).
3. Created `platform/base/flow-analytics/flow-enrichment-tables.yaml` — new GitOps-owned
   ConfigMap embedding both CSV files.
4. Added `app_ports` and `hostname_map` enrichment table definitions to `vector-config.yaml`.
5. Added new `enrich_flow` VRL transform (after `enrich_k8s`) that sets:
   - `dst_app_label` / `dst_app_category` / `dst_app_source` from port registry
   - `src_hostname` / `dst_hostname` from hostname map
   - `src_display_name` / `dst_display_name` (formatted or RFC1918 fallback)
6. Added `enrichment-tables` volume and `/etc/vector/flow-enrichment` mount to
   flow-collector Deployment.
7. Updated sink input from `enrich_k8s` → `enrich_flow`.

**Field source priority for `dst_app_label`:**

- `existing` — set from hardcoded VRL `port_apps` map (pre-existing ports)
- `registry` — set from `app_ports` enrichment table (new ports: Ceph, NFS-mountd, etcd)
- `unlabeled` — no match in either

**VRL caveat:** String concatenation with enrichment table record fields (e.g., `r.short_name + "..."`)
requires `string!(r.short_name)` — VRL 0.45.0's type checker does not verify that
`to_string(r.short_name)` returns `string` when `r` is from a `get_enrichment_table_record` call.
`string!()` has a guaranteed static return type. Discovered during rollout, fixed before go-live.

**Validated (2026-05-02):**

- `kustomize build --enable-helm platform/overlays/lab` passes twice (52 resources)
- ArgoCD: Synced/Healthy (commit 1ad7af1)
- flow-collector: 2/2 Running, 0 restarts
- Vector: clean startup, no errors
- OpenSearch: 9896 new-style flows in 15 min confirmed with new fields
- `dst_app_source` breakdown: existing=3005, registry=1420, unlabeled=5471
- prox2 (172.18.1.20): `src_display_name = "prox2 (172.18.1.20)"`, `src_hostname = "prox2.vgriz.com"` ✓
- NFS-Mountd port 786: `dst_app_label = "NFS-Mountd"`, `dst_app_source = "registry"` ✓
- etcd-Peer port 2380: `dst_app_label = "etcd-Peer"`, `dst_app_source = "registry"` ✓
- Pod IPs (10.244.x.x): `src_display_name = "unknown-10.244.x.x"` (RFC1918 fallback) ✓
- Ceph OSD port 6802: no new flows observed in 15-minute post-restart window (labeling confirmed by port registry entry; will appear when Ceph replication traffic arrives)

### Phase 3a — OpenSearch Mapping Hardening — COMPLETE (2026-05-03)

**What was implemented:**

Updated `platform/base/flow-analytics/opensearch-index-template.yaml` to explicitly
map all 7 new Phase 2 enrichment fields as `keyword`:

- `src_hostname`, `dst_hostname`
- `src_display_name`, `dst_display_name`
- `dst_app_label`, `dst_app_category`, `dst_app_source`

Previously these were dynamically mapped by OpenSearch as `text` with a
`.keyword` subfield (256 char limit). Future daily indices (`flows-YYYY.MM.DD`)
will use native `keyword` with no size limit and no `.keyword` path needed.

**Index mapping caveat:**

The current-day index (`flows-2026.05.03` and any earlier indices) retains
`text + .keyword` dynamic mapping from when Phase 2 first wrote these fields.
Aggregations and dashboards MUST use the `.keyword` path (e.g.,
`dst_app_source.keyword`) until those indices age out of retention. Beginning
2026-05-04, new daily indices will have native `keyword` and plain field paths
will work without `.keyword`.

No rollover was forced. Operators can request a manual rollover to get clean
mapping on today's index if needed (low-risk: today's index is ~3.1M docs).

**Validated (2026-05-03):**

- `kustomize build --enable-helm platform/overlays/lab` passes twice (52 resources)
- ArgoCD: Synced/Healthy (commit 58554bd)
- `PUT /_index_template/flows` → `{"acknowledged":true}`
- Template stored in OpenSearch with all 7 new fields as `{"type":"keyword"}`
- `dst_app_source.keyword` aggregation: unlabeled=1,781,751 existing=937,570 registry=380,745
- `dst_app_label.keyword` top: Kubernetes=526,160 etcd-Peer=259,969 Ceph-Monitor=86,804
- `src_display_name.keyword` top: kube1(172.18.1.61)=506,666 kube3=462,546 kube2=454,699
- `dst_app_category.keyword` top: k8s=804,064 network=180,598 storage=105,327
- Vector: no errors post-restart

### Phase 3b — Workload-Pattern VRL — COMPLETE (2026-05-03)

**What was implemented:**

Extended the `enrich_flow` VRL transform's app-labeling priority chain from
`existing > registry > unlabeled` to `existing > registry > workload > unlabeled`.

In the `else` branch (registry lookup missed), the VRL now checks:

```vrl
dst_wl = to_string(.dst_k8s_workload) ?? ""
src_wl = to_string(.src_k8s_workload) ?? ""
if starts_with(dst_wl, "rook-ceph-osd") || starts_with(src_wl, "rook-ceph-osd") {
  .dst_app_label    = "Ceph-OSD"
  .dst_app_category = "storage"
  .dst_app_source   = "workload"
} else {
  .dst_app_source = "unlabeled"
}
```

**Precedence notes:**

- Ceph OSD traffic on fixed known ports (6800–6804, 6808) already in the registry
  continues to get `dst_app_source = "registry"` — the registry wins because the
  port lookup succeeds before the workload check runs.
- Ephemeral destination ports used in Ceph OSD replication (e.g., 34036, 59604)
  have no registry match; the workload check fires for these.
- Both `dst_k8s_workload` and `src_k8s_workload` are checked so that bidirectional
  flows (e.g., rook-ceph-mon → rook-ceph-osd ephemeral reply) are also captured.

**Validated (2026-05-03, index flows-2026.05.04):**

- `kustomize build --enable-helm platform/overlays/lab` passes twice (52 resources)
- ArgoCD: Synced/Healthy (commit 143db09)
- flow-collector: 2/2 Running, 0 restarts
- Vector: no errors
- `dst_app_source` distribution: unlabeled=102,260 existing=53,913 registry=25,738 **workload=72**
- Workload-source sample:
  - `rook-ceph-mgr-b → rook-ceph-osd-2` dst_port=34036: Ceph-OSD/storage/workload ✓
  - `rook-ceph-mon-i → rook-ceph-osd-0` dst_port=59604: Ceph-OSD/storage/workload ✓
- Registry still wins: etcd-Peer=17,971 Ceph-Monitor=5,804 Ceph-OSD(fixed ports)=1,052 NFS-Mountd=14 ✓
- Existing still wins: Kubernetes=36,594 DNS=11,646 Proxmox=3,297 ✓

**Note on index:** flows-2026.05.04 has native `keyword` mapping for all enrichment
fields (Phase 3a template applied). No `.keyword` path required for aggregations
on this index.

### Phase 4 — Dashboard Label Updates

**Deployed 2026-05-03, commits 3317ca6 and fbd8eb9.**

**What changed:**

| Dashboard | Changes |
|-----------|---------|
| Flow — Top Talkers | Replaced raw src_ip/dst_ip term buckets with nested src_display_name.keyword → src_ip and dst_display_name.keyword → dst_ip. Column headers: Source, Source IP, Destination, Destination IP. Drill-down links pass the raw IP (not display name) so Destination Analysis filtering stays intact. |
| Flow — Destination Analysis | Top Destinations: same nested display_name → ip approach. Top Destination Ports: replaced old `app` bucket with dst_app_label.keyword + dst_app_source.keyword. New panel: App Labels, Categories & Sources — shows dst_app_label / dst_app_category / dst_app_source, making enrichment path visible. Top Sources (Inbound): nested src_display_name → src_ip. |
| Flow — Traffic Mix | Full Breakdown table: replaced old `app` / `protocol_name` nested buckets with dst_app_label.keyword, dst_app_category.keyword, dst_app_source.keyword. Traffic by Application renamed to "Traffic by App Label", updated to use dst_app_label.keyword. New panel: Top App Categories by Volume. New panel: Traffic by Enrichment Source — shows labeled vs unlabeled split. GeoIP notice shifted down. |
| Network Observability — Home | Added GitOps flow enrichment status row (✅ Active). Updated "High-volume unlabeled ports" from 📋 Backlog to ⚠️ Residual (expected). |

**Implementation caveat — .keyword column names:**
The Grafana OpenSearch datasource plugin preserves the `.keyword` suffix in column names
returned from terms aggregations. All `renameByName` keys and `byName` fieldConfig
overrides use the `.keyword` form (e.g., `dst_app_label.keyword`) to match correctly.
Numeric fields (`dst_port`, `src_ip`, `dst_ip`) and metric fields (`Sum`, `Count`) are
unaffected.

**Validated (2026-05-03):**

- `kustomize build --enable-helm platform/overlays/lab` passes twice (deterministic)
- ArgoCD: Synced/Healthy (commit fbd8eb9)
- Flow — Top Talkers: Source column shows `prox2 (172.18.1.20)`, `truenas01 (172.18.1.96)` etc.
  Source IP column shows raw IPs in blue (linked). Drill-down links functional.
- Flow — Traffic Mix: Full Breakdown columns Category | App Label | Port | Enrich Source |
  Total Volume | Flow Count. Top rows: storage/NFS/2049/existing, storage/Ceph-OSD/6802/registry.
  Top App Categories shows k8s, infrastructure, network, storage, web. Traffic by Enrichment
  Source shows unlabeled / existing / registry split.
- Flow — Destination Analysis: Destinations show as `truenas01 (172.18.1.96)` etc.
  App Labels, Categories & Sources panel visible with enrichment path breakdown.
- Network Observability — Home: GitOps flow enrichment row shows ✅ Active.
- No datasource errors. Volumes render as GB. Flow counts render as K notation.

**Evidence screenshots:**

- `docs/evidence/post-wave5-flow-enrichment-phase4/top-talkers.png`
- `docs/evidence/post-wave5-flow-enrichment-phase4/traffic-mix-top.png`
- `docs/evidence/post-wave5-flow-enrichment-phase4/destination-analysis.png`
- `docs/evidence/post-wave5-flow-enrichment-phase4/home-status.png`
- `docs/evidence/post-wave5-flow-enrichment-phase4/top-talkers-table-zoom.png`
- `docs/evidence/post-wave5-flow-enrichment-phase4/traffic-mix-table-zoom.png`

### Phase 5 — Validation and Closeout — COMPLETE (2026-06-08)

Validated on live data via read-only OpenSearch aggregation queries against
`flows-2026.06.08` (last complete day, 3,480,542 docs, native `keyword` mapping).
ArgoCD `network-observability` Synced/Healthy; flow-collector 2/2 Running.

Full evidence: `docs/evidence/post-wave5-flow-enrichment-phase5/final-validation-20260608.txt`

**Check 1 — top unlabeled ports now labeled (by byte volume): PASS**

`dst_app_source` split: existing 232.90 GiB, registry 123.26 GiB, workload 47.14 GiB,
unlabeled 70.45 GiB — **~85% of byte volume is now labeled** (403.3 of 473.8 GiB).
Every top-unlabeled port from the Phase 1 audit now carries a label: 6801/6802
Ceph-OSD, 786 NFS-Mountd, 813/747/741 NFS-Aux, 2380 etcd-Peer, 3300 Ceph-Monitor,
6808 Ceph-Manager. Category rollup is storage-dominated (399.77 GiB), matching the
homelab's Ceph + NFS profile.

**Check 2 — Proxmox/TrueNAS show friendly names: PASS**

`src_display_name` / `dst_display_name` render infrastructure hosts as `name (ip)`:
`prox3 (172.18.1.30)`, `prox2 (172.18.1.20)`, `truenas01 (172.18.1.96)` (228.77 GiB
inbound NFS), all kube nodes. Pod IPs fall back to `unknown-10.244.x.x` as designed.

**Check 3 — no regression in K8s enrichment: PASS**

`src_k8s_type`: node 2,103,390 / external 807,729 / pod 569,389 / internal-unknown
**34** (0.001%). `src_hostname` populated on 79.8% of docs (remainder are pod/public
IPs without a map entry — expected). Phase 3b workload labeling still fires
(`dst_app_source=workload` → Ceph-OSD, 1,835 flows).

**Residual (tracked in backlog, small/optional):** the remaining ~70 GiB unlabeled
slice is dominated by two *identifiable* ports — **5405 Corosync (18.85 GiB)** and
**5353 mDNS (3.06 GiB)** — plus genuinely ephemeral high ports that cannot be
labeled by port. Adding 5405 and 5353 to `app-port-registry.csv` would reclaim
~22 GiB of the residual.

**Validation tooling:** `kustomize build --enable-helm platform/overlays/lab`
passes (no manifest changes in this closeout); `markdownlint-cli2` clean on edited
docs.

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
