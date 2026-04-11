# Flow Ingest and Cutover

## Current State

Proxmox hosts are running softflowd and exporting NetFlow v9/IPFIX to the legacy collector at `172.18.1.207:2055/UDP`. This document covers deploying the in-cluster collector (Wave 2) and the canary cutover to it (Wave 3).

**Known preflight state:**

- prox5 is already exporting NetFlow v9
- Legacy collector is at `172.18.1.207:2055/UDP` (netflow/telegraf stack)
- Canary host default: **prox5**

## Wave 2: Deploy the Flow Analytics Stack

### 1. Enable Wave 2 in the overlay

In `platform/overlays/lab/kustomization.yaml`, uncomment the flow-analytics line:

```yaml
  # Wave 2 — Flow analytics plane (OpenSearch, GoFlow2, Vector)
  - ../../base/flow-analytics
```

Commit and push. ArgoCD syncs the new plane.

### 2. Verify all pods are healthy

```bash
kubectl get pods -n network-observability -l app.kubernetes.io/component=flow-analytics
```

Expected: 2 pods running — `flow-collector` (2/2 containers), `opensearch-0` (1/1).

### 3. Find the flow collector external IP

```bash
kubectl get svc flow-collector -n network-observability
```

Note the `EXTERNAL-IP`. Softflowd on Proxmox hosts will send to this IP on UDP 2055.

### 4. Verify OpenSearch health

```bash
kubectl port-forward -n network-observability svc/opensearch-cluster-master 9200:9200
# In another terminal:
curl -s http://localhost:9200/_cluster/health | python3 -m json.tool
```

Expected: `"status": "green"`.

### 5. Bootstrap OpenSearch configuration

Load the index template:

```bash
curl -X PUT "http://localhost:9200/_index_template/flows" \
  -H "Content-Type: application/json" \
  -d @- < <(kubectl get configmap opensearch-index-template \
    -n network-observability -o jsonpath='{.data.flow-index-template\.json}')
```

Load the ISM retention policy:

```bash
curl -X PUT "http://localhost:9200/_plugins/_ism/policies/flow-retention-14d" \
  -H "Content-Type: application/json" \
  -d @- < <(kubectl get configmap opensearch-ism-policy \
    -n network-observability -o jsonpath='{.data.flow-retention-policy\.json}')
```

Verify:

```bash
curl -s http://localhost:9200/_index_template/flows | python3 -m json.tool
curl -s http://localhost:9200/_plugins/_ism/policies/flow-retention-14d | python3 -m json.tool
```

Both must return the loaded config — not a 404.

### 6. Verify Grafana datasource

In Grafana: **Configuration → Data Sources → OpenSearch Flows → Test**. Expect "Data source is working".

If OpenSearch is healthy but the test fails, check the URL: `http://opensearch-cluster-master:9200` must be resolvable from the Grafana pod. Verify with:

```bash
kubectl exec -n network-observability deploy/grafana -- curl -s http://opensearch-cluster-master:9200/_cluster/health
```

---

## Wave 3: Canary Softflowd Cutover

**Do not proceed until Wave 2 passes Gate 2.**

### Pre-cutover: record existing softflowd targets

SSH to each Proxmox host and record the current destination:

```bash
ssh mikeb@<proxmox-host>
cat /etc/default/softflowd
# or
systemctl cat softflowd | grep -E 'ExecStart|COLLECTOR'
```

Document the current target (expected: `172.18.1.207:2055`) and the softflowd config file location.

### Step 1: Canary — repoint prox5

SSH to prox5 (the canary host):

```bash
ssh mikeb@<prox5-ip>
```

Find the softflowd config location:

```bash
systemctl cat softflowd | grep -A5 ExecStart
# Common locations: /etc/default/softflowd, /etc/softflowd.conf, /etc/softflowd/softflowd.conf
```

Update the collector destination to the flow-collector external IP noted in Wave 2 Step 3:

```bash
# Backup original config
sudo cp /etc/default/softflowd /etc/default/softflowd.bak

# Replace collector address (adjust path if different)
sudo sed -i 's|172.18.1.207:2055|<FLOW_COLLECTOR_IP>:2055|g' /etc/default/softflowd

# Verify the change
grep 2055 /etc/default/softflowd

# Restart softflowd
sudo systemctl restart softflowd
sudo systemctl status softflowd
```

### Step 2: Verify canary flows arrive

Wait 2–3 minutes (a few flow export cycles), then check:

```bash
# Check GoFlow2 is receiving records
kubectl exec -n network-observability deploy/flow-collector -c goflow2 -- tail -5 /flows/flows.jsonl

# Check the file is growing
kubectl exec -n network-observability deploy/flow-collector -c goflow2 -- wc -l /flows/flows.jsonl
```

Check Vector is shipping to OpenSearch:

```bash
kubectl exec -n network-observability deploy/flow-collector -c vector -- curl -s localhost:8686/health
```

Check OpenSearch is indexing:

```bash
# Via port-forward (if still open from Wave 2):
curl -s http://localhost:9200/_cat/indices/flows-*?v
curl -s "http://localhost:9200/flows-*/_count"
```

Expected: index exists with document count > 0 and growing.

Check a sample record for correct field mapping:

```bash
curl -s "http://localhost:9200/flows-*/_search?size=1&pretty" | python3 -m json.tool
```

Verify: `src_ip`, `dst_ip`, `bytes`, `protocol_name`, `exporter` fields are present.

If GeoIP credentials were configured:

```bash
curl -s "http://localhost:9200/flows-*/_search?size=1&pretty&q=dst_country:*" | python3 -m json.tool
```

Verify: `dst_country` field is present in at least some records.

### Step 3: Canary verification period

Let prox5 flow for at least 15 minutes before proceeding. Verify:

- Flow count in OpenSearch is steadily increasing
- No indexing errors in Vector logs: `kubectl logs -n network-observability deploy/flow-collector -c vector --tail=20`
- Grafana "Flow — Top Talkers" dashboard shows data

### Step 4: Repoint remaining Proxmox hosts

If canary is healthy, repeat Steps 1–2 for each remaining Proxmox host running softflowd.

Each host:

```bash
ssh mikeb@<proxmox-host>
sudo cp /etc/default/softflowd /etc/default/softflowd.bak
sudo sed -i 's|172.18.1.207:2055|<FLOW_COLLECTOR_IP>:2055|g' /etc/default/softflowd
sudo systemctl restart softflowd
```

After all hosts are repointed:

- Verify flow count growth rate increases (more exporters = more flows)
- Run `curl -s http://localhost:9200/_cat/indices/flows-*?v` and confirm doc count is growing

---

## GeoIP Bootstrap (Optional but Recommended)

GeoIP enrichment requires a free MaxMind account:

1. Register at maxmind.com (free GeoLite2 account)
2. Generate a license key under Account > Manage License Keys
3. Create the Kubernetes Secret:

   ```bash
   kubectl create secret generic geoip-credentials \
     --namespace network-observability \
     --from-literal=license-key='<your-maxmind-license-key>'
   ```

4. Restart the flow-collector pod to trigger the GeoIP download init container:

   ```bash
   kubectl rollout restart deployment/flow-collector -n network-observability
   ```

Without GeoIP, flows still ingest — country/ASN fields will be absent. The dashboards will show empty country/ASN panels but everything else works.

---

## Rollback: Back to Legacy Collector (172.18.1.207:2055)

### Immediate rollback — single host

```bash
ssh mikeb@<proxmox-host>
sudo cp /etc/default/softflowd.bak /etc/default/softflowd
sudo systemctl restart softflowd
sudo systemctl status softflowd
```

Verify the service started and is pointing back to `172.18.1.207:2055`.

### Full rollback — all hosts

Repeat the above on every Proxmox host that was repointed. The in-cluster flow-collector remains deployed and healthy — it simply stops receiving flows.

**No data loss from rollback:** Flows already indexed in OpenSearch remain. The legacy collector resumes receiving from the hosts. The in-cluster collector can be re-enabled later without data gap impact.

### Disable Wave 2 (if OpenSearch needs to be removed)

Re-comment the `../../base/flow-analytics` line in `platform/overlays/lab/kustomization.yaml`, commit, and push. ArgoCD auto-prune removes all flow-analytics resources.

**Warning:** Disabling Wave 2 deletes the OpenSearch PVC on prune. Any indexed flow data is lost. Only do this if you intend to start fresh.

---

## Monitoring Ingest Health

| Check | Command |
|---|---|
| GoFlow2 receiving | `kubectl exec deploy/flow-collector -c goflow2 -- wc -l /flows/flows.jsonl` |
| Vector healthy | `kubectl exec deploy/flow-collector -c vector -- curl -s localhost:8686/health` |
| Vector logs | `kubectl logs deploy/flow-collector -c vector --tail=20` |
| OpenSearch indices | `curl http://localhost:9200/_cat/indices/flows-*?v` (via port-forward) |
| Index doc count | `curl http://localhost:9200/flows-*/_count` |
| ISM policy status | `curl http://localhost:9200/_plugins/_ism/explain/flows-*` |
| Exporter identity | `curl 'http://localhost:9200/flows-*/_search?size=1&pretty&q=exporter:*'` — verify `exporter` field matches prox5 IP |
