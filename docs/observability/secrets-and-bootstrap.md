# Secrets and Bootstrap

This document lists all Kubernetes Secrets required by the infrastructure telemetry plane and the manual steps to create them.

**No secrets are stored in this repository.** All credentials are managed via Kubernetes Secrets created manually or through a secrets manager.

**Status:** Secrets 1–3 have been created in the `network-observability` namespace. Secret 4 (GeoIP) is optional.

## Required Secrets

### 1. grafana-admin-credentials

**Used by:** Grafana (kube-prometheus-stack)

**Namespace:** `network-observability`

**Keys:**

| Key | Description |
|---|---|
| `admin-user` | Grafana admin username |
| `admin-password` | Grafana admin password |

**Create:**

```bash
kubectl create secret generic grafana-admin-credentials \
  --namespace network-observability \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='<your-password>'
```

### 2. unpoller-credentials

**Used by:** UnPoller deployment

**Namespace:** `network-observability`

**Keys:**

| Key | Description |
|---|---|
| `unifi-url` | UniFi controller URL (e.g., `https://unifi.local:8443`) |
| `unifi-user` | UniFi controller username (read-only local account) |
| `unifi-password` | UniFi controller password |

**Create:**

```bash
kubectl create secret generic unpoller-credentials \
  --namespace network-observability \
  --from-literal=unifi-url='https://unifi.local:8443' \
  --from-literal=unifi-user='unpoller' \
  --from-literal=unifi-password='<your-password>'
```

**Prerequisites:**

- Create a **read-only local account** on the UniFi controller for UnPoller.
  Do not use the admin account.
- The UniFi controller must be network-reachable from the Kubernetes cluster.

### 3. proxmox-credentials

**Used by:** Proxmox VE Exporter deployment

**Namespace:** `network-observability`

**Keys:**

| Key | Description |
|---|---|
| `pve-user` | Proxmox API user (e.g., `monitoring@pve`) |
| `pve-password` | Proxmox API user password or API token |

**Create:**

```bash
kubectl create secret generic proxmox-credentials \
  --namespace network-observability \
  --from-literal=pve-user='monitoring@pve' \
  --from-literal=pve-password='<your-password>'
```

**Prerequisites:**

- Create a dedicated Proxmox user with the **PVEAuditor** role.
  This grants read-only access to node and VM metrics.
- The Proxmox API (`https://proxmox.local:8006`) must be network-reachable from the Kubernetes cluster.

### 4. geoip-credentials (Optional)

**Used by:** Flow collector init container (GeoIP database download)

**Namespace:** `network-observability`

**Keys:**

| Key | Description |
|---|---|
| `license-key` | MaxMind GeoLite2 license key (free account) |

**Create:**

```bash
kubectl create secret generic geoip-credentials \
  --namespace network-observability \
  --from-literal=license-key='<your-maxmind-license-key>'
```

**Prerequisites:**

- Register a free account at [maxmind.com](https://www.maxmind.com/en/geolite2/signup).
- Generate a license key under Account > Manage License Keys.
- Without this secret, flows still ingest — GeoIP fields (country, ASN) will be absent.

### 5. truenas-credentials

**Used by:** `truenas-exporter` (TrueNAS REST API v2.0 polling)

**Namespace:** `network-observability`

**Keys:**

| Key | Description |
|---|---|
| `truenas01-api-key` | API key for TrueNAS01 (172.18.1.96) |
| `truenas02-api-key` | API key for TrueNAS02 (172.18.1.97) |

**Create:**

```bash
kubectl create secret generic truenas-credentials \
  --namespace network-observability \
  --from-literal=truenas01-api-key='<truenas01-api-key>' \
  --from-literal=truenas02-api-key='<truenas02-api-key>'
```

**Prerequisites:**

- Generate an API key per host in the TrueNAS UI (Credentials > Local Users >
  API Keys, or the top-right key icon). A read-only key is sufficient.
- Without this secret, the `truenas-exporter` pod will not start (the keys are
  required env vars). The storage dashboard then has no data.

### 6. ipmi-exporter-config

**Used by:** `ipmi-exporter` (IPMI-over-LAN polling of the Supermicro BMCs)

**Namespace:** `network-observability`

**Keys:**

| Key | Description |
|---|---|
| `ipmi_remote.yml` | Full ipmi_exporter config file, including BMC user/password |

**Create:**

```bash
cat > /tmp/ipmi_remote.yml <<'YAML'
modules:
  default:
    user: "<bmc-user>"
    pass: "<bmc-password>"
    driver: "LAN_2_0"
    privilege: "user"
    timeout: 30000
    collectors:
      - bmc
      - ipmi
      - chassis
      - sel
YAML
kubectl create secret generic ipmi-exporter-config \
  --namespace network-observability \
  --from-file=ipmi_remote.yml=/tmp/ipmi_remote.yml
rm /tmp/ipmi_remote.yml
```

**Prerequisites:**

- The whole config file lives in the Secret because ipmi_exporter reads
  credentials from its config file — keeping the entire file out of git avoids
  any split-source confusion.
- BMC targets are defined in the ServiceMonitor, not in this config
  (proxy-scrape pattern: `/ipmi?target=<bmc-ip>`).
- Without this secret, the `ipmi-exporter` pod will not start (config volume
  mount fails). The Server Hardware dashboard then has no data.

## Bootstrap Order

1. Create the `network-observability` namespace (ArgoCD will do this on first sync, or create it manually):

   ```bash
   kubectl create namespace network-observability
   ```

2. Create all Secrets listed above (grafana, unpoller, proxmox, and optionally geoip).

3. Apply the ArgoCD Application or let ArgoCD sync from git:

   ```bash
   kubectl apply -f platform/apps/network-observability.yaml
   ```

4. Verify all pods reach Ready state:

   ```bash
   kubectl get pods -n network-observability
   ```

5. Access Grafana:

   ```bash
   kubectl port-forward -n network-observability svc/grafana 3000:80
   ```

   Then open `http://localhost:3000` and log in with the Grafana admin credentials.

## SNMP Community String

The SNMP config is inlined in the chart's `config` value (`snmp-exporter-values.yaml`). It contains a placeholder community string (`SNMP_COMMUNITY`). Before deployment:

- **Option A (quick start):** Edit the ConfigMap to replace `SNMP_COMMUNITY` with the actual community string. This is acceptable for a private lab but not recommended for production.
- **Option B (recommended):** Migrate to SNMPv3 with auth credentials stored in a Kubernetes Secret. The snmp_exporter supports `--config.expand-environment-variables` for this pattern.

## ArgoCD Repository Access

The repository is **private**. ArgoCD needs credentials to pull manifests:

```bash
argocd repo add https://github.com/FlunkyMonkey/networkanalyzer.git \
  --username <github-user> \
  --password <github-pat-or-token>
```

Or use an SSH deploy key:

```bash
argocd repo add git@github.com:FlunkyMonkey/networkanalyzer.git \
  --ssh-private-key-path ~/.ssh/deploy_key
```

This must be done before the first ArgoCD sync.

## Network Reachability Checklist

Pods in the cluster must be able to reach:

| Target | Port | Used By |
|---|---|---|
| MikroTik CRS328 | UDP 161 (SNMP) | SNMP Exporter |
| UniFi Controller | TCP 8443 (HTTPS) | UnPoller |
| Proxmox API | TCP 8006 (HTTPS) | PVE Exporter |

| MaxMind GeoIP download | TCP 443 (HTTPS) | Flow collector init container |

Verify network policies or firewall rules allow these connections from the `network-observability` namespace.

Additionally, Proxmox hosts running softflowd must be able to reach the `flow-collector` Service external IP on **UDP 2055**.
See [flow-ingest-and-cutover.md](flow-ingest-and-cutover.md) for cutover details.
