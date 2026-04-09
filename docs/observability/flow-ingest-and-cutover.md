# Flow Ingest and Cutover

## Current State

Softflowd exporters are already running on Proxmox hosts, exporting NetFlow v9/IPFIX to a collector endpoint. This phase deploys the in-cluster collector and requires repointing softflowd to the new destination.

## Cutover Steps

### 1. Deploy the Flow Analytics Plane

Ensure the ArgoCD sync has deployed all flow-analytics components and pods are healthy:

```bash
kubectl get pods -n network-observability -l app.kubernetes.io/component=flow-analytics
```

### 2. Find the Flow Collector Service IP

The `flow-collector` Service is type `LoadBalancer`. MetalLB will assign an external IP:

```bash
kubectl get svc flow-collector -n network-observability
```

Note the `EXTERNAL-IP` value. This is the IP softflowd should send flows to.

### 3. Repoint Softflowd on Each Proxmox Host

SSH to each Proxmox host and update the softflowd configuration:

```bash
# Check current softflowd config
cat /etc/default/softflowd

# Update the collector destination
# Replace COLLECTOR_IP with the flow-collector Service external IP
sudo sed -i 's/FLOW_COLLECTOR=.*/FLOW_COLLECTOR="COLLECTOR_IP:2055"/' /etc/default/softflowd

# Restart softflowd
sudo systemctl restart softflowd
```

Repeat for each Proxmox host running softflowd.

### 4. Verify Flows Are Arriving

Check that GoFlow2 is receiving flows:

```bash
# Check the flow data file is growing
kubectl exec -n network-observability deploy/flow-collector -c goflow2 -- ls -la /flows/flows.jsonl

# Tail recent flow records
kubectl exec -n network-observability deploy/flow-collector -c goflow2 -- tail -5 /flows/flows.jsonl
```

Check that Vector is shipping to OpenSearch:

```bash
# Check Vector health
kubectl exec -n network-observability deploy/flow-collector -c vector -- curl -s localhost:8686/health

# Check OpenSearch for flow indices
kubectl exec -n network-observability deploy/flow-collector -c vector -- \
  curl -s "http://opensearch-cluster-master:9200/_cat/indices/flows-*?v"
```

### 5. Access OpenSearch Dashboards

```bash
kubectl port-forward -n network-observability svc/opensearch-dashboards 5601:5601
```

Open `http://localhost:5601`. Create the `flows-*` index pattern if not auto-imported.

## Bootstrap: OpenSearch Configuration

After OpenSearch is running, load the index template and ISM policy:

### Load Index Template

```bash
# From a pod with access to OpenSearch, or via port-forward
kubectl port-forward -n network-observability svc/opensearch-cluster-master 9200:9200

# In another terminal:
curl -X PUT "http://localhost:9200/_index_template/flows" \
  -H "Content-Type: application/json" \
  -d @- < <(kubectl get configmap opensearch-index-template \
    -n network-observability -o jsonpath='{.data.flow-index-template\.json}')
```

### Load ISM Policy

```bash
curl -X PUT "http://localhost:9200/_plugins/_ism/policies/flow-retention-30d" \
  -H "Content-Type: application/json" \
  -d @- < <(kubectl get configmap opensearch-ism-policy \
    -n network-observability -o jsonpath='{.data.flow-retention-policy\.json}')
```

### Import Dashboard Objects

```bash
# Port-forward to Dashboards
kubectl port-forward -n network-observability svc/opensearch-dashboards 5601:5601

# Import index pattern
curl -X POST "http://localhost:5601/api/saved_objects/_import" \
  -H "osd-xsrf: true" \
  --form file=@- < <(kubectl get configmap opensearch-dashboards-objects \
    -n network-observability -o jsonpath='{.data.flow-index-pattern\.ndjson}')

# Import saved searches
curl -X POST "http://localhost:5601/api/saved_objects/_import" \
  -H "osd-xsrf: true" \
  --form file=@- < <(kubectl get configmap opensearch-dashboards-objects \
    -n network-observability -o jsonpath='{.data.flow-saved-searches\.ndjson}')
```

## GeoIP Bootstrap (Optional but Recommended)

GeoIP enrichment requires a free MaxMind account:

1. Register at [maxmind.com](https://www.maxmind.com/en/geolite2/signup)
1. Generate a license key under Account > Manage License Keys
1. Create the Kubernetes Secret:

   ```bash
   kubectl create secret generic geoip-credentials \
     --namespace network-observability \
     --from-literal=license-key='<your-maxmind-license-key>'
   ```

1. Restart the flow-collector pod to trigger the GeoIP download init container:

   ```bash
   kubectl rollout restart deployment/flow-collector -n network-observability
   ```

Without GeoIP, flows still ingest and index — country/ASN fields will simply be absent.

## Rollback

If flows need to be sent back to the original collector:

1. SSH to each Proxmox host
2. Restore the original softflowd collector destination
3. Restart softflowd

The in-cluster flow-collector will stop receiving flows but remains deployed and healthy. No data loss occurs — existing indexed flows remain in OpenSearch.

## Monitoring Ingest Health

| Check | Command |
|---|---|
| GoFlow2 receiving | `kubectl exec deploy/flow-collector -c goflow2 -- wc -l /flows/flows.jsonl` |
| Vector healthy | `kubectl exec deploy/flow-collector -c vector -- curl -s localhost:8686/health` |
| OpenSearch indices | `curl http://localhost:9200/_cat/indices/flows-*?v` (via port-forward) |
| Index doc count | `curl http://localhost:9200/flows-*/_count` |
| ISM policy status | `curl http://localhost:9200/_plugins/_ism/explain/flows-*` |
