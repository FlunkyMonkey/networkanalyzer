# Wave 2 Canary Cutover Plan

## Overview

Wave 3 uses a canary-first approach to repoint softflowd exporters from the legacy collector to the in-cluster GoFlow2 collector. The canary host is **prox5**, which is already exporting NetFlow v9.

**Legacy rollback target:** `172.18.1.207:2055/UDP` (existing netflow/telegraf collector)

---

## Pre-conditions (must all be true before starting)

| # | Condition | How to verify |
|---|---|---|
| C1 | Wave 2 pods are healthy | `kubectl get pods -n network-observability -l app.kubernetes.io/component=flow-analytics` — all Running |
| C2 | OpenSearch cluster is green | `curl localhost:9200/_cluster/health` — status: green |
| C3 | Flow collector has external IP | `kubectl get svc flow-collector -n network-observability` — EXTERNAL-IP assigned |
| C4 | Index template loaded | `curl localhost:9200/_index_template/flows` — returns template, not 404 |
| C5 | ISM policy loaded | `curl localhost:9200/_plugins/_ism/policies/flow-retention-14d` — returns policy |
| C6 | Grafana datasource works | Grafana → Data Sources → OpenSearch Flows → Test |
| C7 | Legacy target recorded | `/etc/default/softflowd` on each host documents `172.18.1.207:2055` |

Do not start cutover if any condition fails.

---

## Step-by-Step Canary Procedure

### Phase A: Canary (prox5 only)

**1. Record current prox5 softflowd target**

```bash
ssh mikeb@<prox5-ip>
cat /etc/default/softflowd
# Note the exact COLLECTOR value — must be 172.18.1.207:2055
```

**2. Repoint prox5 to in-cluster collector**

```bash
# On prox5:
FLOW_IP=<FLOW_COLLECTOR_EXTERNAL_IP>   # from kubectl get svc

sudo cp /etc/default/softflowd /etc/default/softflowd.bak
sudo sed -i "s|172.18.1.207:2055|${FLOW_IP}:2055|g" /etc/default/softflowd
grep 2055 /etc/default/softflowd        # confirm substitution
sudo systemctl restart softflowd
sudo systemctl status softflowd         # confirm active/running
```

**3. Wait for first flow export cycle (2–3 minutes)**

Softflowd typically exports every 60–300 seconds depending on configuration. Wait at least one full cycle.

**4. Verify flows arrive**

```bash
# Check GoFlow2 file is non-empty
kubectl exec -n network-observability deploy/flow-collector -c goflow2 -- wc -l /flows/flows.jsonl

# Tail recent records — must show prox5's IP in "SamplerAddress" or "exporter" field
kubectl exec -n network-observability deploy/flow-collector -c goflow2 -- tail -3 /flows/flows.jsonl

# Confirm Vector is healthy
kubectl exec -n network-observability deploy/flow-collector -c vector -- curl -s localhost:8686/health
```

**5. Verify OpenSearch indexing**

```bash
# Port-forward (if not already open)
kubectl port-forward -n network-observability svc/opensearch-cluster-master 9200:9200 &

curl -s "http://localhost:9200/_cat/indices/flows-*?v"
curl -s "http://localhost:9200/flows-*/_count"
```

Expected: at least one `flows-YYYY.MM.DD` index, count > 0.

**6. Verify field correctness**

```bash
curl -s "http://localhost:9200/flows-*/_search?size=1&pretty" | python3 -m json.tool
```

Confirm these fields are present and non-null:

- `src_ip` — must be a valid IP (not "0.0.0.0")
- `dst_ip` — must be a valid IP
- `bytes` — must be > 0
- `protocol_name` — must be TCP, UDP, ICMP, or GRE
- `exporter` — must match prox5's IP address

**7. Canary soak period: 15 minutes minimum**

Monitor during soak:

```bash
# Watch doc count grow
watch -n 30 'curl -s http://localhost:9200/flows-*/_count'

# Watch Vector logs for errors
kubectl logs -n network-observability deploy/flow-collector -c vector --tail=20 -f
```

**Canary pass criteria:**

- [ ] Doc count is growing steadily
- [ ] No ERROR lines in Vector logs
- [ ] Flow fields are correctly mapped
- [ ] `exporter` field matches prox5

If any criterion fails → **Canary Rollback** (see below), investigate before retrying.

---

### Phase B: Full Cutover (remaining hosts)

After canary passes, proceed host by host. For each remaining Proxmox host:

```bash
ssh mikeb@<host-ip>
FLOW_IP=<FLOW_COLLECTOR_EXTERNAL_IP>

# Backup
sudo cp /etc/default/softflowd /etc/default/softflowd.bak

# Repoint
sudo sed -i "s|172.18.1.207:2055|${FLOW_IP}:2055|g" /etc/default/softflowd
grep 2055 /etc/default/softflowd

# Restart
sudo systemctl restart softflowd
sudo systemctl status softflowd
```

Verify OpenSearch doc count increases after each host is added.

**Final verification after all hosts:**

```bash
# Check total doc count is growing
curl -s "http://localhost:9200/flows-*/_count"

# Check exporters present — one per active Proxmox host
curl -s "http://localhost:9200/flows-*/_search?size=0&pretty" -H "Content-Type: application/json" -d '{
  "aggs": { "exporters": { "terms": { "field": "exporter", "size": 20 } } }
}'
```

Expected: one bucket per active Proxmox host.

---

## Rollback Procedures

### Canary Rollback (prox5 only)

```bash
ssh mikeb@<prox5-ip>
sudo cp /etc/default/softflowd.bak /etc/default/softflowd
sudo systemctl restart softflowd
sudo systemctl status softflowd
grep 2055 /etc/default/softflowd   # confirm back to 172.18.1.207:2055
```

Legacy collector resumes receiving from prox5. In-cluster collector remains running but idle.

### Full Rollback (all hosts)

Repeat canary rollback on every host that was repointed. Order doesn't matter.

**After full rollback:**

- Verify legacy collector at `172.18.1.207:2055` is receiving from all hosts (check legacy stack health)
- Wave 2 stack (OpenSearch + flow-collector) remains deployed and can be retried

### Wave 2 Stack Removal (last resort)

Only if OpenSearch itself needs to be decommissioned:

1. Perform full softflowd rollback above
2. Re-comment `../../base/flow-analytics` in `platform/overlays/lab/kustomization.yaml`
3. Commit and push
4. ArgoCD auto-prune removes flow resources and deletes the PVC

**Data loss warning:** Removing the PVC deletes all indexed flow data.

---

## Stop/Go Decision Matrix

| Observation | Action |
|---|---|
| GoFlow2 file empty after 5 minutes | Stop — check firewall/routing from prox5 to FLOW_COLLECTOR_IP:2055/UDP |
| GoFlow2 file growing, Vector errors in logs | Stop — check OpenSearch connectivity from flow-collector pod |
| OpenSearch indexing, but field mapping errors | Stop — re-apply index template, check schema |
| All checks pass at canary | Continue to Phase B |
| Doc count not growing after Phase B | Check each host's softflowd status individually |
| Wave 1 metrics degraded | Stop — investigate Prometheus scrape health before continuing |

---

## Timing Reference

| Phase | Expected duration |
|---|---|
| Wave 2 pod startup (OpenSearch) | 3–7 minutes (JVM init + PVC mount) |
| Bootstrap (index template + ISM) | < 5 minutes |
| Canary repoint + first flow export | 2–3 minutes |
| Canary soak period | 15 minutes minimum |
| Per-host full cutover | 2 minutes each |
| Total elapsed (typical) | 30–60 minutes |
