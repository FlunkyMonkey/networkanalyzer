# Backlog

Future-wave items not in scope for the current rollout. These are tracked here so they aren't lost but do not block active waves.

## Wave 2 Hardening Items

These items were accepted as residual conditions at Wave 2 closeout. They must be
addressed before Wave 2 is considered fully hardened, but they do not block Wave 3
planning.

### Suppress or route GoFlow2 template-warning noise

GoFlow2 logs `No template found for ...` warnings on every undecodable flow until
the NetFlow v9 template packet arrives. At scale these warnings obscure actionable
log output.

**Action:** Filter or route these log lines to a lower severity level, or configure
GoFlow2 structured logging to suppress the template-missing class. Evaluate whether
a Vector log transform can drop them before they reach persistent storage.

### Verify ISM retention policy is applying to `flows-*`

The `flow-retention-14d` ISM policy was loaded at bootstrap, but correct attachment
to the `flows-*` index pattern should be confirmed after 14+ days of operation.

**Action:** After 14 days, check that old indices are rolling off:

```bash
# port-forward to OpenSearch
curl localhost:9200/_cat/indices/flows-*?v&s=index
curl localhost:9200/_plugins/_ism/explain/flows-$(date -d '15 days ago' +%Y.%m.%d)
```

### Clean remaining GitOps drift and local-only changes

`vector-config` and related resources were patched directly against the cluster
during Wave 2 debugging. ArgoCD may show drift until a clean sync cycle runs.

**Action:** After the next ArgoCD hard refresh and sync, verify `argocd app get
network-observability` shows Synced, Healthy with no diff. Resolve any lingering
out-of-sync resources.

### Validate Grafana Top Talkers performance on 100k+ documents

The Top Talkers dashboard uses a `terms` aggregation over `flows-*`. At 116k+
documents and growing, query latency should be measured.

**Action:** Open Flow — Top Talkers in Grafana over a 24h window and time the
response. If > 5s, evaluate increasing `number_of_shards` on the index template
or adding a `date_histogram` pre-filter.

### Standardize explicit Proxmox exporter service pattern across all hosts

The explicit `softflowd-netflow.service` custom unit pattern was developed during
Wave 2 rollout. This should be documented and standardized so new Proxmox hosts
added in the future follow the same pattern.

**Action:** Add a host-provisioning checklist to `flow-ingest-and-cutover.md`
covering: mask default unit, configure `/etc/default/softflowd`, deploy
`softflowd-netflow.service`.

### Document interface autodetect to avoid hardcoded NIC mistakes

Different Proxmox hosts use different interface names (`eno1`, `eth0`, `bond0`, etc.).
Hardcoding the interface in the service unit causes silent failures on hosts with
different NICs.

**Action:** Update `flow-ingest-and-cutover.md` with the interface autodetect
pattern (grep from `/etc/default/softflowd`) and add a warning against hardcoding
interface names in the service unit template.

## Server Hardware Monitoring

**Scope:** Physical host telemetry for Proxmox nodes and storage hosts.

**Candidate sources:**

- IPMI / BMC sensors (via ipmi_exporter or freeipmi-based collectors)
- Redfish API (modern BMC REST interface — vendor-dependent)
- node_exporter textfile collectors (custom scripts writing .prom files)
- Vendor-specific exporters (Dell iDRAC, HP iLO, Supermicro)

**Signals:**

- CPU and board temperatures
- Fan speeds and status
- PSU voltage, wattage, and failure state
- ECC memory errors and alerts
- Disk health (SMART via node_exporter or smartctl_exporter)
- Chassis intrusion alarms
- Overall hardware health rollup

**Why it matters:** Proxmox node failures from thermal, PSU, or disk issues are invisible to Kubernetes-level monitoring. Hardware telemetry provides early warning before a node goes down.

**Not in Wave 2.** This is a telemetry plane extension, not a flow plane concern. Candidate for a future wave after the flow plane is stable.

## Network Switch Hardware Monitoring

**Scope:** Hardware health for MikroTik CRS328 and, if worthwhile, UniFi switches.

**MikroTik CRS328 signals (via SNMP):**

- Board temperature (`/system/health` OIDs)
- System temperature
- Fan status (if applicable to the CRS328 model)
- PSU state
- PoE budget and per-port PoE consumption
- SFP/transceiver optical levels (TX/RX power, temperature)
- Hardware alarms

**UniFi switch signals (via controller API):**

- Device temperature (if exposed by UnPoller)
- Fan status
- PoE budget and per-port draw
- Uplink SFP health

**Why it matters:** The current SNMP exporter collects interface traffic counters but not hardware health. A switch overheating or losing a PSU would not be visible until ports go down.

**Implementation notes:**

- MikroTik hardware OIDs may require extending the SNMP exporter config with additional MIBs (MIKROTIK-MIB health subtree)
- UnPoller may already expose some device health metrics — check `unpoller_device_*` metric families
- PoE monitoring is operationally valuable for tracking AP and camera power budgets

**Not in Wave 2.** This extends the existing SNMP/UnPoller integration. Candidate for a future telemetry plane enhancement.
