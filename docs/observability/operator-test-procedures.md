# Operator Test Procedures

Detailed test cases for the operator to execute after each rollout wave. These verify the platform meets functional requirements.

## T1. Interface Utilization

**When to run:** After Wave 1.

**Procedure:**

1. Open Grafana → Switch Interface Utilization dashboard (`/d/infra-iface-util`)
1. Verify at least one MikroTik switch interface shows non-zero bandwidth
1. Check that `ifDescr` labels show recognizable port names (e.g., ether1, sfp1)
1. Verify the Utilization % gauge shows a percentage between 0–100 for active ports
1. Change time range to "Last 6 hours" — verify historical data exists
1. Check that inactive ports show 0 bps (not missing data)

**Pass criteria:** Live per-port bandwidth data is visible. Historical data extends to at least the polling start time.

---

## T2. WiFi Client Bandwidth

**When to run:** After Wave 1.

**Procedure:**

1. Open Grafana → UniFi AP & WLAN Clients dashboard (`/d/infra-unifi-clients`)
1. Verify at least one WiFi client shows RX/TX bandwidth
1. Check that client names and MACs are visible in the legend
1. Verify the "Connected WLAN Clients" stat shows a plausible number
1. Generate traffic from a known WiFi device — verify bandwidth increases in the chart
1. Check AP Status table — verify APs show uptime values

**Pass criteria:** Per-client bandwidth is visible with client identity. Count matches expected connected clients.

---

## T3. Proxmox VM Bandwidth

**When to run:** After Wave 1.

**Procedure:**

1. Open Grafana → Proxmox Node & VM Network dashboard (`/d/infra-proxmox-net`)
1. Verify at least one VM shows non-zero network in/out
1. Check that VM names and IDs are visible
1. Verify node-level network is visible
1. Generate traffic from a known VM — verify the chart responds

**Pass criteria:** Per-VM and per-node bandwidth is visible with correct VM names.

---

## T4. Top Talkers

**When to run:** After Wave 3 (softflowd cutover).

**Procedure:**

1. Open Grafana → Homepage (`/d/correlation-home`)
1. Locate the "Top Talkers" table (card 1)
1. Verify it shows source IPs ranked by bytes
1. Verify "Unique Destinations" column shows a count for each source
1. Click a source IP — verify it opens Entity Investigation or OpenSearch
1. Compare the top talker with known heavy users (e.g., a backup server, media server)

**Pass criteria:** Top talkers table shows real IPs with plausible byte counts. Links work.

---

## T5. Destination Analysis

**When to run:** After Wave 3.

**Procedure:**

1. On the Homepage, enter a known server IP in the "Source IP" variable
1. Check the Destinations table (card 2)
1. Verify it shows destination IPs, ports, bytes, and (if GeoIP configured) countries
1. Verify the "App" column shows values like HTTPS, DNS, SSH for well-known ports
1. Check if destination ASN/org is populated (requires GeoIP)

**Pass criteria:** Per-source destination breakdown is visible with port and country data.

---

## T6. Ports, Countries, Apps

**When to run:** After Wave 3.

**Procedure:**

1. On the Homepage, check the "Top Countries" pie chart
1. Verify it shows country names with byte volumes (requires GeoIP)
1. Check the "Top Apps/Services" bar gauge
1. Verify it shows app names (HTTPS, DNS, SSH, etc.) with byte volumes
1. In Grafana → Flow — Traffic Mix dashboard: verify "Top Destination Ports" table includes port 443
1. In Grafana → Flow — Traffic Mix dashboard: verify "Traffic by Destination Country" pie includes expected countries (if GeoIP enabled)

**Pass criteria:** Country and app breakdowns reflect real traffic. Grafana flow dashboards return expected results.

**Note:** If GeoIP was not configured, country fields will be absent. Document this as a known limitation, not a failure.

---

## T7. Hubble UI and K8s Visibility

**When to run:** After Wave 4.

**Procedure:**

1. Port-forward to Hubble UI: `kubectl port-forward svc/hubble-ui 12000:80 -n network-observability`
1. Open `http://localhost:12000`
1. Verify the service map shows at least some pods communicating
1. Select the `network-observability` namespace — verify platform pods appear
1. Select another namespace with known workloads — verify flows are visible
1. Look for DNS flows (queries to CoreDNS)

**Pass criteria:** Hubble UI is connected, shows pod flows, supports namespace filtering.

**Note:** L7 visibility (HTTP methods, paths) requires CiliumNetworkPolicy or pod annotations. L3/L4 visibility is the baseline expectation.

---

## T8. Homepage Order and Drill-Downs

**When to run:** After Wave 5.

**Procedure:**

1. Open Grafana Homepage (`/d/correlation-home`)
1. Verify card order: (1) Top Talkers, (2) Destinations, (3) WiFi Client Bandwidth
1. Verify the freshness bar at the top shows green/healthy indicators
1. Click an IP in Top Talkers → verify Entity Investigation opens with the IP pre-filled
1. In Entity Investigation, verify: outbound flow table, inbound flow table, flow timeline
1. In Entity Investigation, click "View flows for this IP" link — verify Grafana Flow — Destination Analysis opens with IP pre-filtered
1. Click "Open in Hubble UI" link — verify Hubble UI opens
1. Navigate to Investigation Playbooks — verify all 6 playbooks are listed
1. Click the "Back to Home" link on any sub-dashboard — verify it returns to the homepage

**Pass criteria:** Locked card order is correct. All navigation links work. Cross-plane drill-down is functional.

---

## T9. Degraded-State Behavior

**When to run:** After Wave 5.

**Procedure:**

1. Scale down the UnPoller deployment: `kubectl scale deploy/unpoller --replicas=0 -n network-observability`
1. Wait 2 minutes for scrape failure to register
1. Check the Homepage freshness bar — verify "Source Freshness" shows DEGRADED (red)
1. Check Platform Health dashboard — verify UnPoller target shows DOWN
1. Scale UnPoller back up: `kubectl scale deploy/unpoller --replicas=1 -n network-observability`
1. Wait for recovery — verify freshness bar returns to HEALTHY

**Pass criteria:** Degraded state is visually surfaced on the homepage. Recovery is reflected.

---

## T10. Softflowd Cutover Verification

**When to run:** During Wave 3.

**Procedure:**

1. Before cutover, record the current flow count: `curl localhost:9200/flows-*/_count`
1. Repoint softflowd on one host (see [flow-ingest-and-cutover.md](flow-ingest-and-cutover.md))
1. Wait 2 minutes
1. Re-check flow count — verify it has increased
1. Inspect a sample flow: `curl 'localhost:9200/flows-*/_search?size=1&pretty'`
1. Verify fields: `src_ip`, `dst_ip`, `src_port`, `dst_port`, `bytes`, `packets`, `protocol_name`
1. If GeoIP configured, verify: `dst_country`, `dst_asn`, `dst_as_org`
1. Repoint remaining hosts and re-verify count growth rate

**Pass criteria:** Flows are arriving, correctly parsed, enriched (if GeoIP configured), and indexed.

---

## T11. OpenSearch Bootstrap Verification

**When to run:** During Wave 2.

**Procedure:**

1. Port-forward to OpenSearch: `kubectl port-forward svc/opensearch-cluster-master 9200:9200 -n network-observability`
1. Verify cluster health: `curl localhost:9200/_cluster/health?pretty`
   - Expected: `"status": "green"`, `"number_of_nodes": 1`
1. Verify index template: `curl localhost:9200/_index_template/flows`
   - Expected: template with `flows-*` pattern
1. Verify ISM policy: `curl localhost:9200/_plugins/_ism/policies/flow-retention-14d`
   - Expected: policy with 14-day transition to delete
1. Verify Grafana datasource: Grafana → Configuration → Data Sources → "OpenSearch Flows" → Test
   - Expected: "Data source is working"
1. Verify flow dashboards loaded: Grafana → Dashboards → Browse
   - Expected: `flow-top-talkers`, `flow-destinations`, `flow-traffic-mix` all present

**Pass criteria:** OpenSearch is healthy, index template is loaded, ISM policy `flow-retention-14d` is active, Grafana flow datasource test passes, flow dashboards are visible.
