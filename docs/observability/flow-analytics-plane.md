# Flow Analytics Plane

## Component Stack

| Component | Version | Purpose |
|---|---|---|
| GoFlow2 | 2.3.1 | NetFlow v5/v9/IPFIX receiver, outputs JSON |
| Vector | 0.45.0 | Flow enrichment (GeoIP, ASN, app mapping), ships to OpenSearch |
| OpenSearch | 3.6.0 | Indexed flow storage with ISM retention |
| OpenSearch Dashboards | 3.6.0 | Specialist flow investigation UI |

### Why This Stack

- **GoFlow2** is the cleanest open-source NetFlow receiver — lightweight, container-native, no Java dependency. Receives flows on UDP 2055 and outputs structured JSON.
- **Vector** bridges GoFlow2 to OpenSearch while providing built-in GeoIP/ASN enrichment and well-known port-to-app mapping. Avoids adding Kafka (overkill for homelab scale).
- **OpenSearch** provides full-text and structured search over flow data with ISM (Index State Management) for automated 30-day retention.
- **OpenSearch Dashboards** provides the specialist drill-down UI for flow investigation — top talkers, destination analysis, country breakdowns, port/protocol pivots.
- **ElastiFlow** was considered but rejected — the open-source community edition is effectively dead; meaningful enrichment is paywalled.

### Architecture

```text
softflowd (Proxmox hosts)
  │ NetFlow v9/IPFIX, UDP 2055
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
         ┌──────────────────┐
         │ OpenSearch        │
         │ Dashboards (UI)   │
         └──────────────────┘
```

GoFlow2 and Vector run as sidecars in a single pod. GoFlow2 writes JSON lines to a file on a shared emptyDir volume. Vector tails the file, applies enrichment transforms, and bulk-indexes into OpenSearch.

## Ingestion Path

1. Softflowd on each Proxmox host exports NetFlow v9/IPFIX to the `flow-collector` Service on UDP 2055.
2. The Service is type `LoadBalancer` — MetalLB assigns a cluster-reachable IP.
3. GoFlow2 receives flows and writes JSON-line records to `/flows/flows.jsonl`.
4. Vector reads from that file, parses JSON, normalizes field names, enriches with GeoIP/ASN, maps well-known ports to app names, and indexes into OpenSearch as daily indices (`flows-YYYY.MM.DD`).

## Enrichment

### Available Now (Phase 4)

| Enrichment | Source | Coverage |
|---|---|---|
| GeoIP (country, city, lat/lon) | MaxMind GeoLite2-City | Requires free MaxMind account + license key |
| ASN / AS organization | MaxMind GeoLite2-ASN | Same license key |
| App / service name | Well-known port mapping in Vector config | ~20 common services (HTTP, HTTPS, DNS, SSH, etc.) |
| Protocol name | Protocol number mapping in Vector config | TCP, UDP, ICMP, GRE |

### Deferred (Future Phases)

| Enrichment | Source | Status |
|---|---|---|
| DNS name (IP → domain) | DNS query logs | DNS logging source not yet wired (open decision from Phase 1) |
| App/category (deep) | Firewalla classification or nDPI | Firewalla API feasibility not yet validated |
| Threat intelligence | External feeds | Phase 7 scope |

## What This Phase Can Answer

- **Top talkers** — which IPs are transferring the most data?
- **Destinations** — what IPs/ports is a given server connecting to?
- **Countries** — where is traffic going geographically? (requires GeoIP)
- **Apps/services** — what well-known services dominate traffic?
- **Ports/protocols** — what ports and protocols are in use?
- **Time-range queries** — how did traffic patterns change over the last 30 days?

## What This Phase Cannot Answer Yet

- **Domain names** — only IP addresses, not DNS names (needs DNS enrichment)
- **Deep app/category** — only well-known port mapping, not DPI-level classification
- **Per-WLAN-client flow breakdown** — softflowd sees bridge-level traffic, not per-WiFi-client
- **Kubernetes pod-level flows** — Phase 5 scope
- **Firewalla-enriched flows** — Firewalla API integration deferred

## Storage and Retention

- OpenSearch stores flows in daily indices: `flows-YYYY.MM.DD`
- ISM policy `flow-retention-30d` automatically deletes indices older than 30 days
- Single shard, zero replicas (appropriate for homelab single-node)
- 50Gi PVC on Ceph storage
- Index template enforces correct field mappings and types

## Known Blind Spots

1. **No DNS enrichment yet** — flow records show destination IPs but not domain names
2. **Port-based app mapping only** — HTTPS traffic on port 443 is all labeled "HTTPS" regardless of actual site/service
3. **Softflowd bridge-level visibility** — VM-to-VM traffic on the same host may not be distinguished cleanly
4. **No sampling compensation** — if softflowd uses sampling, byte/packet counts may undercount
5. **GeoIP requires MaxMind account** — without it, country/ASN fields are absent but flows still index
6. **Single-node OpenSearch** — no replication; node failure means flow data is unavailable until restored

## Self-Health

- Vector exposes health at `/health` on port 8686 (liveness + readiness probes configured)
- GoFlow2 liveness is checked via file existence
- OpenSearch and Dashboards have probes via their Helm charts
- Vector metrics are available at the `/api` endpoint for future Prometheus scraping

## Firewalla Integration Path (Future)

Firewalla is not integrated in this phase. When API feasibility is validated:

- Firewalla flow logs could be ingested via a separate Vector source (HTTP or file-based)
- Firewalla's native app/category classification could supplement the port-based mapping
- DNS query logs from Firewalla could provide IP → domain enrichment
- A dedicated `firewalla-collector` container or sidecar would be added to the flow-collector pod or as a separate deployment
