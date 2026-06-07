# Configuration

## Table of Contents

- [Prometheus Monitoring Stack](#prometheus-monitoring-stack)
- [Architecture](#architecture)
- [Files](#files)
- [Prerequisites](#prerequisites)
- [Deployment](#deployment)
  - [1. Create Discord Webhooks](#1-create-discord-webhooks)
  - [2. Create the Discord Webhook Secrets](#2-create-the-discord-webhook-secrets)
  - [3. Provision Grafana Admin Credentials](#3-provision-grafana-admin-credentials)
  - [4. Apply Manifests](#4-apply-manifests)
  - [5. Verify](#5-verify)
  - [6. Test End-to-End](#6-test-end-to-end)
- [Alert Routing](#alert-routing)
  - [Route Priority](#route-priority)
  - [Grouping](#grouping)
  - [Inhibit Rules](#inhibit-rules)
- [Custom Alert Rules](#custom-alert-rules)
  - [Scope](#scope)
  - [Current Alerts](#current-alerts)
  - [Adding New Alerts](#adding-new-alerts)
  - [Default Rules](#default-rules)
  - [Disabled Service Monitors](#disabled-service-monitors)
- [Alertmanager-Discord Adapter](#alertmanager-discord-adapter)
  - [Resource Usage](#resource-usage)
- [Security Hardening](#security-hardening)
  - [Service Account Token Mounting](#service-account-token-mounting)
  - [Pod and Container Security Context](#pod-and-container-security-context)
  - [Verifying Security Controls](#verifying-security-controls)
- [Troubleshooting](#troubleshooting)
  - [Alerts are Firing in Grafana/Prometheus but not appearing in Discord](#alerts-are-firing-in-grafanaprometheus-but-not-appearing-in-discord)
  - [Alerts stuck in Pending state](#alerts-stuck-in-pending-state)
  - ["Failed to unpack inbound alert request" in adapter logs](#failed-to-unpack-inbound-alert-request-in-adapter-logs)
  - [Alerts going to the wrong channel](#alerts-going-to-the-wrong-channel)
  - [Verifying Alertmanager config](#verifying-alertmanager-config)
  - [Viewing active alerts in Alertmanager](#viewing-active-alerts-in-alertmanager)
- [Custom Values Configuration](#custom-values-configuration)
- [Prometheus PVC Size, Retention Time and Size](#prometheus-pvc-size-retention-time-and-size)
  - [Prometheus Admin API](#prometheus-admin-api)

## Prometheus Monitoring Stack

Kube-Prometheus-Stack deployment on RKE2 with real-time Discord alerting.

**Helm Chart**: [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

## Architecture

```
Prometheus (rule evaluation)
    │
    ▼
Alertmanager (routing, grouping, inhibition)
    │
    ├─ Watchdog/InfoInhibitor ──▶  null (silenced)
    ├─ severity=info          ──▶  null (silenced)
    ├─ severity=critical      ──▶  alertmanager-discord-critical  ──▶  #alerts-critical
    ├─ severity=warning       ──▶  alertmanager-discord-warning   ──▶  #alerts-warning
    └─ everything else        ──▶  alertmanager-discord-default   ──▶  #alerts-default
```

Grafana is not exposed externally to reduce the attack surface. Alerting flows through Alertmanager and Discord instead.

To open the Grafana dashboard without exposing it publicly, reach it through the **Rancher dashboard → Service Discovery → Services**: select the cluster, open the `monitoring` namespace, find the `prometheus-grafana` Service, and use its **HTTP** port link to open the UI proxied through your authenticated Rancher session. (Equivalently, `kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80` and browse to `http://localhost:3000`.)

## Files

| File | Description |
|------|-------------|
| `values.yaml` | Helm values for kube-prometheus-stack — includes Alertmanager routing config, receiver definitions, and inhibit rules |
| `alertmanager-discord.yaml` | Deployments and Services for the [alertmanager-discord](https://github.com/benjojo/alertmanager-discord) adapter (one per Discord channel) |
| `discord-webhook-secrets.yaml` | **Template** showing the shape of the three Discord webhook Secrets — created imperatively, not applied directly (see Step 2) |
| `prometheus-rules.yaml` | Custom `PrometheusRule` CRD with alert definitions for pods, deployments, containers, nodes, and PVCs |
| `monitoring-storage-class.yaml` | Longhorn StorageClass used by Prometheus, Alertmanager, and Grafana PVCs |

## Prerequisites

- RKE2 cluster with Longhorn storage driver
- `monitoring` namespace created
- Helm 3.x with the `prometheus-community` repo added:
  ```bash
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm repo update
  ```

## Deployment

### 1. Create Discord Webhooks

In Discord, go to **Server Settings > Integrations > Webhooks** and create three webhooks — one per channel:

- `#alerts-default`
- `#alerts-critical`
- `#alerts-warning`

### 2. Create the Discord Webhook Secrets

> The `discord-webhook-secrets.yaml` file in this directory is a **template** showing the shape of the three Secrets. Don't edit it with real URLs or apply it directly — create the Secrets imperatively so the webhook URLs never touch the repo. `kubectl create secret` base64-encodes values for you.

Create one Secret per channel, passing the webhook URL from Discord directly:

```bash
kubectl create secret generic discord-webhook-default  -n monitoring \
  --from-literal=DISCORD_WEBHOOK='https://discord.com/api/webhooks/YOUR_ID/YOUR_TOKEN'

kubectl create secret generic discord-webhook-critical -n monitoring \
  --from-literal=DISCORD_WEBHOOK='https://discord.com/api/webhooks/YOUR_ID/YOUR_TOKEN'

kubectl create secret generic discord-webhook-warning  -n monitoring \
  --from-literal=DISCORD_WEBHOOK='https://discord.com/api/webhooks/YOUR_ID/YOUR_TOKEN'
```

### 3. Provision Grafana Admin Credentials

> Note: Execute these in the `prometheus-grafana` directory where the `values.yaml` is located. The file names must match the ones specified in the grafana section of the `values.yaml`.

```bash
# echo the creds into their respective files
echo -n '<username>' > ./admin-user
echo -n '<strong-password>' > ./admin-password
```

```bash
# confirm the creds
cat admin-user
cat admin-password
```

```bash
# apply & confirm the secret
kubectl create secret generic grafana-admin-credentials --from-file=./admin-user --from-file=admin-password -n monitoring
kubectl describe secret -n monitoring grafana-admin-credentials
kubectl get secret -n monitoring grafana-admin-credentials -o jsonpath="{.data.admin-user}" | base64 --decode
kubectl get secret -n monitoring grafana-admin-credentials -o jsonpath="{.data.admin-password}" | base64 --decode
```

```bash
# remove the creds files once verified
rm admin-user && rm admin-password
```

### 4. Apply Manifests

```bash
# Storage class (if not already created)
kubectl apply -f monitoring-storage-class.yaml

# Alertmanager-discord adapter deployments and services
# (Discord webhook secrets were already created imperatively in step 2)
kubectl apply -f alertmanager-discord.yaml

# Custom PrometheusRules
kubectl apply -f prometheus-rules.yaml

# Install or upgrade the helm release
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace -f values.yaml
```

### 5. Verify

```bash
# Check adapter pods are running
kubectl get pods -n monitoring -l app=alertmanager-discord

# Confirm Alertmanager loaded the config
kubectl get secret -n monitoring alertmanager-prometheus-alertmanager \
  -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d

# Confirm PrometheusRules are loaded
kubectl get prometheusrules -n monitoring

# Port-forward to Alertmanager UI to inspect routes
kubectl port-forward -n monitoring svc/alertmanager 9093:9093
# Visit http://localhost:9093/#/status
```

### 6. Test End-to-End

Fire a test alert directly to Alertmanager:

```bash
kubectl exec -n monitoring $(kubectl get pod -n monitoring \
  -l app.kubernetes.io/name=alertmanager -o jsonpath='{.items[0].metadata.name}') -- \
  wget -q -O- \
  --post-data='[{"labels":{"alertname":"TestAlert","severity":"warning","namespace":"test-ns"},"annotations":{"summary":"Test alert in **test-ns**","description":"Verifying Discord pipeline for namespace **test-ns**."}}]' \
  --header="Content-Type: application/json" \
  http://localhost:9093/api/v2/alerts
```

The alert should appear in `#alerts-warning` within ~30 seconds.

## Alert Routing

### Route Priority

Routes are evaluated top-to-bottom. The first match wins (`continue: false`):

1. **Watchdog / InfoInhibitor** → `null` receiver (silenced, never sent to Discord)
2. **severity=info** → `null` receiver (silenced — catches default chart alerts like the built-in CPUThrottlingHigh that duplicate custom rules)
3. **severity=critical** → `discord-critical` → `#alerts-critical`
4. **severity=warning** → `discord-warning` → `#alerts-warning`
5. **Everything else** → `discord-default` → `#alerts-default`

### Grouping

Alerts are grouped by `alertname`, `namespace`, and `job`. This means multiple instances of the same alert in the same namespace are batched into a single Discord message rather than flooding the channel.

| Parameter | Value | Effect |
|-----------|-------|--------|
| `group_wait` | 30s | Wait 30s after first alert in a new group before sending |
| `group_interval` | 5m | Wait 5m before sending updates for an existing group |
| `repeat_interval` | 2h | Re-send a firing alert every 2 hours |

### Inhibit Rules

| Source | Suppresses | Condition |
|--------|-----------|-----------|
| `severity=critical` | `severity=warning` | Same `alertname` and `namespace` |
| `alertname=InfoInhibitor` | `severity=info` | Same `namespace` |

**Important**: All `severity=info` alerts are routed to the `null` receiver and will never reach Discord. Use `severity=warning` or higher for any alert that must be delivered.

## Custom Alert Rules

Custom alerts are defined in `prometheus-rules.yaml` as a `PrometheusRule` CRD. Prometheus picks these up automatically because `ruleSelectorNilUsesHelmValues: false` is set in `values.yaml`.

### Scope

All custom alert rules apply **cluster-wide across all namespaces**. The PromQL expressions have no namespace filter, so any pod, deployment, node, or PVC in the cluster is covered. To scope an alert to a specific namespace, add a label matcher (e.g., `{namespace="my-app"}`).

### Current Alerts

| Alert | Severity | Scope | Condition | `for` Duration |
|-------|----------|-------|-----------|----------------|
| CPUThrottlingHigh | warning | All namespaces | CPU throttling > 25% | 15m |
| PodCrashLooping | critical | All namespaces | > 3 restarts in 15m | 5m |
| PodNotReady | warning | All namespaces | Pod not ready | 10m |
| DeploymentReplicasMismatch | warning | All namespaces | Desired != ready replicas | 10m |
| ContainerHighCPU | warning | All namespaces | CPU > 90% of limit | 10m |
| ContainerOOMKilled | critical | All namespaces | Container OOM terminated and restarted within last 10m | 0m (instant, auto-resolves) |
| NodeHighMemoryUsage | critical | All nodes | Node memory > 90% | 5m |
| NodeHighDiskUsage | warning | All nodes | Disk usage > 85% | 10m |
| PVCAlmostFull | warning | All namespaces | PVC > 85% full | 10m |

The `for` duration means the condition must be continuously true for that period before the alert transitions from **Pending** to **Firing**. Only **Firing** alerts are sent to Discord.

In addition to these custom rules, the kube-prometheus-stack `defaultRules` provide cluster-wide coverage for etcd, kubelet, kube-apiserver, kube-scheduler, node-exporter, and other core components.

### Adding New Alerts

Append a new rule to `prometheus-rules.yaml` or create a separate `PrometheusRule` manifest. Use `**{{ $labels.namespace }}**` in annotations for bold namespace rendering in Discord:

```yaml
- alert: MyAppHighLatency
  expr: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{namespace="my-app"}[5m])) by (le)) > 2
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "HighLatency in **{{ $labels.namespace }}**"
    description: "P99 latency in namespace **{{ $labels.namespace }}** is {{ $value }}s."
```

Then apply:

```bash
kubectl apply -f prometheus-rules.yaml
```

No Helm upgrade is needed — Prometheus watches for PrometheusRule changes automatically.

### Default Rules

The kube-prometheus-stack chart deploys a standard set of alert rules (etcd, kubelet, node-exporter, etc.). These are controlled by the `defaultRules.rules.*` flags in `values.yaml`. Some default rules overlap with custom rules in `prometheus-rules.yaml` (e.g., `CPUThrottlingHigh`). The default versions use `severity=info` and are silenced by the `severity=info → null` route, so only the custom versions with Discord-formatted annotations and `severity=warning` are delivered.

### Disabled Service Monitors

The following components are disabled because RKE2 binds them to `127.0.0.1` by default, making them unreachable by Prometheus:

| Component | Why disabled |
|-----------|-------------|
| `kubeControllerManager` | Binds to 127.0.0.1 on RKE2 — causes 100% TargetDown alerts |
| `kubeScheduler` | Binds to 127.0.0.1 on RKE2 — causes 100% TargetDown alerts |
| `kubeProxy` | Binds to 127.0.0.1 on RKE2 — causes 100% TargetDown alerts |

Their corresponding default alert rules (`kubeControllerManager`, `kubeProxy`, `kubeSchedulerAlerting`, `kubeSchedulerRecording`) are also disabled since there is no scrape data to evaluate against.

Control plane visibility is maintained through `kubeApiServer` and `kubeEtcd`, which are the most critical components. Scheduler and controller-manager failures typically surface through pod scheduling issues and etcd metrics. If direct metrics are needed later, the RKE2 server config can be updated to bind these components to the node IP instead of localhost.

## Alertmanager-Discord Adapter

Each adapter instance is a lightweight Go binary ([benjojo/alertmanager-discord](https://github.com/benjojo/alertmanager-discord)) that:

1. Listens on port 9094 for Alertmanager webhook POSTs
2. Translates the payload into Discord embed format
3. POSTs to a single Discord webhook URL (one channel per instance)

Three instances run — one per Discord channel. Each reads its webhook URL from a Kubernetes Secret.

### Resource Usage

Each adapter instance requests 50m CPU / 32Mi memory with a 64Mi memory limit. Total for all three: 150m CPU / 96Mi memory.

## Security Hardening

### Service Account Token Mounting

Components that don't interact with the K8s API have `automountServiceAccountToken` disabled to reduce the attack surface. This is configured explicitly in `values.yaml` for every component.

**Note**: For components managed by the Prometheus Operator (Alertmanager, Prometheus), the token mount must be disabled at **both** the `serviceAccount` level and the pod spec level (e.g., `alertmanagerSpec.automountServiceAccountToken`), since the operator generates the pod spec independently of the ServiceAccount default.

| Component | `automountServiceAccountToken` | Reason |
|-----------|-------------------------------|--------|
| **Prometheus** | `true` | Service discovery and scraping targets via K8s API |
| **Prometheus Operator** | `true` | Manages CRDs (ServiceMonitor, PrometheusRule, etc.) |
| **kube-state-metrics** | `true` | Queries K8s API for cluster state metrics |
| **Grafana** | `true` (`autoMount`) | Sidecar watches ConfigMaps for dashboard provisioning |
| **Alertmanager** | `false` (serviceAccount + alertmanagerSpec) | Receives alerts from Prometheus over HTTP, sends to webhooks — no K8s API interaction |
| **node-exporter** | `false` | Reads host metrics from /proc and /sys only |
| **alertmanager-discord** | `false` | Translates webhooks to Discord — outbound HTTPS only |

### Pod and Container Security Context

All components are hardened with explicit `securityContext` settings in `values.yaml`. The level of hardening varies based on each component's requirements:

| Component | `runAsNonRoot` | `readOnlyRootFilesystem` | `drop ALL` caps | `seccompProfile` | Notes |
|-----------|:-:|:-:|:-:|:-:|-------|
| **Prometheus** | Yes (65534) | Yes | Yes | RuntimeDefault | Full hardening — writes only to PVC |
| **Alertmanager** | Yes (65534) | Yes | Yes | RuntimeDefault | Full hardening — writes only to PVC |
| **Prometheus Operator** | Yes (65534) | Yes | Yes | RuntimeDefault | Full hardening — stateless controller |
| **kube-state-metrics** | Yes (65534) | Yes | Yes | RuntimeDefault | Full hardening — read-only API queries |
| **Grafana** | Yes (472) | No | Yes | RuntimeDefault | Partial — sidecar/plugins need writable /tmp and /var/lib/grafana. emptyDir mounted at /tmp |
| **node-exporter** | Yes (65534) | Yes | Yes | RuntimeDefault | Full hardening — accesses host /proc and /sys via hostPath mounts (readable without root) |
| **alertmanager-discord** | Yes (65534) | Yes | Yes | RuntimeDefault | Full hardening — dedicated SA, outbound HTTPS only |

### Verifying Security Controls

**Pod-level security context** (runAsUser, runAsNonRoot, seccomp):

```bash
kubectl get pods -n monitoring -o custom-columns=\
'POD:.metadata.name,RUN_AS_USER:.spec.securityContext.runAsUser,RUN_AS_NON_ROOT:.spec.securityContext.runAsNonRoot,SECCOMP:.spec.securityContext.seccompProfile.type'
```

**Service account token mounting**:

```bash
kubectl get pods -n monitoring -o custom-columns=\
'POD:.metadata.name,AUTOMOUNT:.spec.automountServiceAccountToken'
```

**Running user/group IDs**:

```bash
kubectl get pods -n monitoring -o custom-columns=\
'POD:.metadata.name,UID:.spec.securityContext.runAsUser,GID:.spec.securityContext.runAsGroup,FSGROUP:.spec.securityContext.fsGroup'
```

**Check for privileged containers** (should return no matches):

```bash
kubectl get pods -n monitoring -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}{": privileged="}{.securityContext.privileged}{"\n"}{end}{end}' | grep "true"
```

**Full detail on a specific pod** (look for the Security Context sections):

```bash
kubectl describe pod -n monitoring -l app.kubernetes.io/name=prometheus | grep -A10 "Security Context"
```

## Troubleshooting

### Alerts are Firing in Grafana/Prometheus but not appearing in Discord

**Check if the alert was already sent recently.** Alertmanager deduplicates notifications — it won't re-send the same alert group until `repeat_interval` (2h) expires. To force re-send:

```bash
kubectl delete pod -n monitoring -l app.kubernetes.io/name=alertmanager
```

This is safe — silences and state are persisted. Only the in-memory notification dedup cache resets.

### Alerts stuck in Pending state

The alert condition is true but the `for` duration hasn't elapsed yet. For example, `CPUThrottlingHigh` requires 15 consecutive minutes above 25% throttling before it fires. If the condition fluctuates, it resets.

### "Failed to unpack inbound alert request" in adapter logs

These are harmless. They come from liveness/readiness probe `GET` requests — the adapter tries to JSON-decode every request and logs an error for GETs with no body. Actual alert delivery uses `POST`.

### Alerts going to the wrong channel

Check the `severity` label on the alert. The route order is: null (Watchdog/InfoInhibitor) > null (severity=info) > critical > warning > default. An alert without a severity label falls through to `discord-default`.

### Verifying Alertmanager config

```bash
kubectl get secret -n monitoring alertmanager-prometheus-alertmanager \
  -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d
```

### Viewing active alerts in Alertmanager

```bash
kubectl exec -n monitoring $(kubectl get pod -n monitoring \
  -l app.kubernetes.io/name=alertmanager -o jsonpath='{.items[0].metadata.name}') -- \
  wget -q -O- http://localhost:9093/api/v2/alerts | python3 -m json.tool
```


---

## Custom Values Configuration

The **"retentionSize"** attribute/settings in the custom **values.yaml** should be set lower than the storage class PVC size. For example, if the storage class PVC size is "25Gi" then the retentionSize should be below "25Gi". Which can be set to something like "15GiB".

This would prevent any bottleneck when the PVC size is about to be exceeded and free up space. 

At the same time, disable **"multipathd"** service on all nodes before installing **Longhorn** to prevent a mount failed warning/exception such as below, when pods restart or crash:
> Note: **Multipathd** combines multiple storage paths to the same disk/LUN into one logical device for redundancy and HA.

```
MountVolume.MountDevice failed for volume "pvc-275f1d05-1735-4236-bab6-xxxxxx" : rpc error: code = Internal desc = format of disk "/dev/longhorn/pvc-275f1d05-1735-4236-bab6-xxxxxx" failed: type:("ext4") target:("/var/lib/kubelet/plugins/kubernetes.io/csi/driver.longhorn.io/efd3eb006c0c703ceeb49e2778194d04891c7b1ff42ed79837140db4dbf03515/globalmount") options:("defaults") errcode:(exit status 1) output:(mke2fs 1.47.0 (5-Feb-2023)
```

Use the following command to disable **"multipathd"** service on all nodes:
> Note: run these commands on all nodes.

```
# confirm multipathd is active and running
systemctl status multipathd

# check whether it is in use and if device mapper multipath devices exist
## if the output is empty then it means it's probably not in use
multipath -ll
ls /dev/mapper | grep -E '^mpath|^3600'

# stop and disable multipathd service
systemctl stop multipathd.socket
systemctl stop multipathd.service
systemctl disable multipathd.socket
systemctl disable multipathd.service

# verify multipathd is disabled and inactive
systemctl status multipathd.socket multipathd.service --no-pager
systemctl status multipathd
```

If **"multipathd"** service is required later down the line, i.e, for Storage Area Network (SAN), it should be enabled and configured properly to prevent the **multipath daemon from adding additional block devices created by Longhorn**. Refer to longhorn's official guide to do so:
> Note: go through the github issue below if disabling multipathd service doesn't fix the issue related to MountDevice failed warning/error. 

- [Troubleshooting: `MountVolume.SetUp failed for volume` due to multipathd on the node](https://longhorn.io/kb/troubleshooting-volume-with-multipath/)
- [BUG - MountVolume.SetUp failed for volume ~ rpc error: code = Internal desc = exit status 1 due to multipathd on the node #1210](https://github.com/longhorn/longhorn/issues/1210)


## Prometheus PVC Size, Retention Time and Size

In production, decrease the PVC size to **5Gi or 10Gi** and the **"retentionSize"** should be lower than the PVC size. The retention time setting/attribute uses different units, i.e, days (d), weeks (w), hour (h), etc. Refer to prometheus configuration [guide](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) for the different units supported for `YAML`.

### Prometheus Admin API

The **"enableAdminAPI"** attribute/settings must be set to **"false"**. Setting it to **"true"** enables **dangerous administrative endpoints** on the Prometheus web API that can modify or delete Time Series Database (TSDB) data, and this increases the attack surface.

If it must be set to **"true"**, handle with care and ensure it's disabled when it's no longer needed. Note that these Admin API endpoints are for advanced users and prometheus works fine without enabling it.

Refer to prometheus [documentation](https://prometheus.io/docs/prometheus/latest/querying/api/#tsdb-admin-apis) for more information on it. Additionally, see prometheus's security [guide](https://prometheus.io/docs/operating/security/) for security best practices.
