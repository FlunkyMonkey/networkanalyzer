# Backlog

Future-wave items not in scope for the current rollout. These are tracked here so they aren't lost but do not block active waves.

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
