# Flow Analytics Plane

## Component Stack

| Component | Version | Purpose |
|---|---|---|
| GoFlow2 | 2.3.1 | NetFlow v5/v9/IPFIX receiver, outputs JSON |
| Vector | 0.45.0 | Flow enrichment (GeoIP, ASN, app mapping), ships to OpenSearch |
| OpenSearch | 3.6.0 chart | Indexed flow storage with ISM retention |
| Grafana (existing) | 10.5.15 | Visualization UI — sole flow UI surface |

**OpenSearch Dashboards is NOT deployed.** Grafana with the `grafana-opensearch-datasource` plugin is the only UI. This avoids running a second web app and keeps investigation within the single platform Grafana.

### Why This Stack

- **GoFlow2** is the cleanest open-source NetFlow receiver — lightweight, container-native, no Java dependency. Receives flows on UDP 2055 and outputs structured JSON.
- **Vector** bridges GoFlow2 to OpenSearch while providing built-in GeoIP/ASN enrichment and well-known port-to-app mapping. Avoids adding Kafka (overkill for homelab scale).
- **OpenSearch** provides full-text and structured search over flow data with ISM (Index State Management) for automated 14-day retention.
- **Grafana** is already deployed as the platform UI; adding OpenSearch as a datasource keeps all investigation in one surface.
- **ElastiFlow** was considered but rejected — the open-source community edition is effectively dead; meaningful enrichment is paywalled.
- **OpenSearch Dashboards** was removed from scope — Grafana covers the required investigation workflows.

### Architecture

```text
softflowd (Proxmox hosts)
  │ NetFlow v9/IPFIX, UDP 2055
  ▼
flow-collector Service (MetalLB LoadBalancer)
  │
  ▼
┌──────────────────────────────────┐
│ flow-collector pod               │
│ ┌──────────┐   ┌───────────────┐ │
│ │ GoFlow2  │──▶│ Vector        │ │
│ │ (receive)│   │ (enrich+ship) │ │
│ └──────────┘   └───────────────┘ │
│     shared emptyDir volume       │
└──────────────────────────────────┘
                   │ HTTP bulk index
                   ▼
            ┌─────────────┐
            │  OpenSearch  │
            │  (storage)   │
            └─────────────┘
                   │
                   ▼
         ┌────────────────────────────┐
         │ Grafana (existing)         │
         │ + opensearch-flows DS      │
         │ + flow dashboards          │
         └────────────────────────────┘
```

GoFlow2 and Vector run as sidecars in a single pod. GoFlow2 writes JSON lines to a file on a shared emptyDir volume. Vector tails the file, applies enrichment transforms, and bulk-indexes into OpenSearch as daily indices (`flows-YYYY.MM.DD`).

### OpenSearch Node Placement

OpenSearch is pinned by nodeAffinity preference:

- **kube4** (weight 100) — primary placement
- **kube2** (weight 50) — fallback

`preferredDuringSchedulingIgnoredDuringExecution` is used — the scheduler honors the preference but will place elsewhere if neither node has capacity. This avoids a hard requirement while keeping the StatefulSet predictable.

## Ingestion Path

1. Softflowd on each Proxmox host exports NetFlow v9/IPFIX to the `flow-collector` Service on UDP 2055.
2. The Service is type `LoadBalancer` — MetalLB assigns a cluster-reachable IP.
3. GoFlow2 receives flows and writes JSON-line records to `/flows/flows.jsonl`.
4. Vector reads from that file, parses JSON, normalizes field names, enriches with GeoIP/ASN, maps well-known ports to app names, and indexes into OpenSearch as daily indices (`flows-YYYY.MM.DD`).

### Canary Cutover Model

Wave 3 uses a canary approach:

1. Deploy Wave 2 stack (this wave) and verify OpenSearch health.
2. Repoint **prox5 only** to the new collector IP (canary).
3. Verify flows arrive and index correctly.
4. If healthy for 15+ minutes, repoint remaining Proxmox hosts.
5. Legacy collector at `172.18.1.207:2055` remains running — rollback is `sed` + `systemctl restart`.

See [wave2-canary-cutover.md](wave2-canary-cutover.md) for the full cutover runbook.

## Enrichment

### Available Now (Wave 2)

| Enrichment | Source | Coverage |
|---|---|---|
| GeoIP (country, city, lat/lon) | MaxMind GeoLite2-City | Requires free MaxMind account + license key |
| ASN / AS organization | MaxMind GeoLite2-ASN | Same license key |
| App / service name | Well-known port mapping in Vector config | ~20 common services (HTTP, HTTPS, DNS, SSH, etc.) |
| Protocol name | Protocol number mapping in Vector config | TCP, UDP, ICMP, GRE |

### Deferred (Future Waves)

| Enrichment | Source | Status |
|---|---|---|
| DNS name (IP → domain) | DNS query logs | DNS logging source not yet wired |
| App/category (deep) | Firewalla classification or nDPI | Firewalla API feasibility not validated |
| Threat intelligence | External feeds | Future wave scope |

## What This Plane Can Answer

- **Top talkers** — which IPs are transferring the most data?
- **Destinations** — what IPs/ports is a given server connecting to?
- **Countries** — where is traffic going geographically? (requires GeoIP)
- **Apps/services** — what well-known services dominate traffic?
- **Ports/protocols** — what ports and protocols are in use?
- **Time-range queries** — how did traffic patterns change over the last 14 days?

## Dashboards

Three dashboards are provisioned in Grafana via the `grafana-flow-dashboards` ConfigMap:

| Dashboard | UID | Purpose |
|---|---|---|
| Flow — Top Talkers | `flow-top-talkers` | Top src/dst IPs by bytes; total volume timeseries |
| Flow — Destination Analysis | `flow-destinations` | Per-host destination breakdown; country pivot |
| Flow — Traffic Mix | `flow-traffic-mix` | App, protocol, country distribution; top ports; top ASNs |

**Datasource:** `OpenSearch Flows` (UID: `opensearch-flows`), provisioned via `grafana-flow-datasource` ConfigMap. Grafana's sidecar auto-loads it when the ConfigMap is present.

## Storage and Retention

- OpenSearch stores flows in daily indices: `flows-YYYY.MM.DD`
- ISM policy `flow-retention-14d` automatically deletes indices older than 14 days
- Single shard, zero replicas (appropriate for homelab single-node)
- 50Gi PVC on Ceph (`rook-ceph-block`)
- Index template enforces correct field mappings and types

### Resource Sizing Note (Homelab)

At typical homelab flow volumes (prox5 + a few hosts):

- **Estimated flow rate:** 1,000–10,000 flows/minute at steady state
- **Estimated index size:** ~1–3 KB per flow record (post-enrichment, with GeoIP)
- **14-day storage estimate:** ~20–85 GB at 10k flows/min; well within the 50Gi PVC
- **JVM heap:** 2Gi (`-Xms2g -Xmx2g`) is appropriate for this scale
- **If flow volume is lower** (< 1k flows/min): the 50Gi PVC will be very lightly used
- **Retention note:** 14d is day-1 target; 30d was the architectural target but is deferred — address if Ceph capacity allows after data is collected

## Field / Schema Contract

The Vector transform produces normalized fields written to OpenSearch. GoFlow2 raw fields → normalized canonical names:

| OpenSearch Field | GoFlow2 Source | Type | Notes |
|---|---|---|---|
| `@timestamp` | injected by Vector | date | ingest time |
| `flow_start` | `TimeFlowStart` | date | epoch seconds |
| `flow_end` | `TimeFlowEnd` | date | epoch seconds |
| `src_ip` | `SrcAddr` | ip | |
| `dst_ip` | `DstAddr` | ip | |
| `src_port` | `SrcPort` | integer | |
| `dst_port` | `DstPort` | integer | |
| `protocol` | `Proto` | integer | protocol number |
| `protocol_name` | derived | keyword | TCP/UDP/ICMP/GRE |
| `bytes` | `Bytes` | long | |
| `packets` | `Packets` | long | |
| `tcp_flags` | `TCPFlags` | integer | |
| `exporter` | `SamplerAddress` | ip | which host sent the flow |
| `flow_type` | `Type` | keyword | NetFlow/IPFIX |
| `app` | derived from dst_port | keyword | HTTP/HTTPS/DNS/SSH etc. |
| `dst_country` | MaxMind GeoIP | keyword | optional |
| `dst_country_code` | MaxMind GeoIP | keyword | optional |
| `dst_city` | MaxMind GeoIP | keyword | optional |
| `dst_latitude` | MaxMind GeoIP | float | optional |
| `dst_longitude` | MaxMind GeoIP | float | optional |
| `dst_asn` | MaxMind GeoIP | long | optional |
| `dst_as_org` | MaxMind GeoIP | keyword | optional |
| `src_country` | MaxMind GeoIP | keyword | optional |
| `src_country_code` | MaxMind GeoIP | keyword | optional |
| `src_asn` | MaxMind GeoIP | long | optional |
| `src_as_org` | MaxMind GeoIP | keyword | optional |

Fields marked "optional" are absent if MaxMind GeoIP credentials are not configured. Flows still index without them.

## Security Stance (Canary Stage)

**OpenSearch security plugin is disabled** (`plugins.security.disabled: true`). Access to OpenSearch is unauthenticated in-cluster HTTP only.

| Aspect | Current State | Risk Level |
|---|---|---|
| Authentication | None — all in-cluster HTTP requests are accepted | Acceptable for canary (in-cluster only) |
| Authorization | None — any pod in the cluster can read/write all indices | Low risk while canary-only; must harden before expansion |
| Encryption in transit | None — HTTP, not HTTPS | Acceptable for in-cluster only |
| Network exposure | No Ingress, no NodePort — accessible only within the cluster | Mitigates external exposure |

**Constraints:** This stance is explicitly limited to the canary stage. Flow data contains network telemetry (source/destination IPs, byte counts, connection metadata) which may have sensitivity.

**Future hardening required before production:**

- Enable OpenSearch Security plugin with TLS + basic auth or OIDC
- Restrict network access to OpenSearch pod via Kubernetes NetworkPolicy
- Rotate from HTTP to HTTPS for the Vector → OpenSearch sink
- Update `grafana-flow-datasource.yaml` to pass credentials via a Secret

This is tracked as a future hardening item. No deployment to production environments until hardening is complete.

## Known Blind Spots

1. **No DNS enrichment yet** — flow records show destination IPs but not domain names
2. **Port-based app mapping only** — HTTPS traffic on port 443 is all labeled "HTTPS" regardless of actual site/service
3. **Softflowd bridge-level visibility** — VM-to-VM traffic on the same host may not be distinguished cleanly
4. **No sampling compensation** — if softflowd uses sampling, byte/packet counts may undercount
5. **GeoIP requires MaxMind account** — without it, country/ASN fields are absent but flows still index
6. **Single-node OpenSearch** — no replication; node failure means flow data is unavailable until restored

## Self-Health

- Vector exposes health at `/health` on port 8686 (liveness + readiness probes configured)
- GoFlow2 liveness is checked via file existence on the shared volume
- OpenSearch has probes via the Helm chart
- OpenSearch cluster health endpoint: `/_cluster/health`
- Grafana datasource health is testable via Grafana UI (Data Sources → Test)
