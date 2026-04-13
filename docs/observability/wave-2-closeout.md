# Wave 2 Closeout

## Executive Summary

Wave 2 — Flow Analytics Plane — is accepted as operational with conditions.

All Proxmox hosts have been migrated to the new flow collector. OpenSearch is indexing flows. Grafana flow datasource is connected. Final document count confirms sustained ingest. Accepted residual conditions are documented below and tracked in the backlog.

Wave 3 planning may begin.

---

## Final Functional Outcome

| Item | Result |
|---|---|
| flow-collector pod | 2/2 Running, healthy |
| OpenSearch | Green, indexing |
| Grafana flow datasource | Connected, "Data source is working" |
| Final document count | 116,134 documents in `flows-*` |
| Timestamp field (`@timestamp`) | Fresh — matching current time |
| Protocol integer mapping | Correct — `protocol` field is integer, `protocol_name` is string label |
| All Proxmox hosts migrated | Yes — all hosts pointing to `172.18.1.200:2055` |

---

## Final Host Rollout Pattern

All Proxmox hosts were migrated using the following pattern:

### Disable legacy paths before activating new ones

```bash
# Stop the package-installed default unit if present
sudo systemctl stop softflowd || true
sudo systemctl disable softflowd || true
sudo systemctl mask softflowd || true

# Stop any active instance on the default path
sudo systemctl stop softflowd@default || true
```

### Activate the explicit custom unit

```bash
# Start the explicit per-host service unit
sudo systemctl start softflowd-netflow.service
sudo systemctl enable softflowd-netflow.service
sudo systemctl status softflowd-netflow.service
```

### Interface autodetect from `/etc/default/softflowd`

Each host's interface was derived from the existing `/etc/default/softflowd` config
rather than hardcoded. This avoids incorrect NIC assumptions across different hosts
(e.g. `eno1` vs `eth0` vs `bond0`).

```bash
# Extract interface from existing config
IFACE=$(grep -oP '(?<=-i )\S+' /etc/default/softflowd)
echo "Using interface: $IFACE"
```

See [operations-notes.md](operations-notes.md) for the rationale behind this pattern.

---

## VRL Fix History

The Vector transform required three patch cycles before flow ingest was stable.
All fixes are committed in the repo and validated against `timberio/vector:0.45.0-distroless-static`.

| Commit | Fix |
|---|---|
| `c6ab432` | GoFlow2 image tag (`v2.3.1` → `v2`), OpenSearch JVM opts duplicate env var |
| `58d0b46` | VRL runtime: `to_timestamp` undefined, `else-if` chain syntax, GeoIP table crash on empty mmdb |
| `b205bf3` | GoFlow2 v2 field contract: snake_case fields, nanosecond timestamps, `api_version: "v8"` |
| `704a29b` | Protocol field: `proto` string label → `protocol_name` + integer reverse-map, graceful JSON parse |

---

## Accepted Residual Conditions

### GoFlow2 NetFlow v9 template warnings

GoFlow2 logs repeated warnings of the form:

```text
No template found for ...
```

This occurs because NetFlow v9 requires the exporter to send a template packet
before data packets can be decoded. GoFlow2 receives data packets before the
template arrives and logs a warning per undecodable flow.

**Impact:** Some flows are dropped at startup or when templates expire and
re-arrive. Steady-state ingest is unaffected once templates are established.
**Status:** Non-blocking. Tracked in backlog for suppression/routing.

### GitOps / ArgoCD cleanliness

The ConfigMap `vector-config` and related resources were patched directly against
the cluster during debugging. ArgoCD may show drift until the next sync applies
the committed state cleanly.

**Required action before hardening complete:** Verify ArgoCD sync is clean
(`argocd app get network-observability` → Synced, Healthy). If not, a hard refresh
and sync cycle will resolve it.
**Status:** Accepted for operational baseline. Tracked in backlog.

---

## Formal Sign-Off

**Wave 2 is accepted as the operational baseline.**

Criteria met:

- All Proxmox hosts exporting to the new collector
- Flow data indexed in OpenSearch with correct field types
- Grafana flow datasource and dashboards functional
- `116,134` documents confirming sustained ingest
- Protocol and timestamp fields validated in live documents

**Wave 3 planning may begin.**

Wave 3 scope: Kubernetes visibility (Hubble UI). Prerequisite: Cilium CNI migration
in `k8s-lab.git` must be completed first. See [rollout-checklists.md](rollout-checklists.md)
Wave 4 and [cni-migration-and-rollout.md](cni-migration-and-rollout.md).
