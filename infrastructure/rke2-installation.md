# RKE2 Installation Guide

This guide documents the step-by-step process for installing RKE2 on control plane and worker nodes, configuring the cluster, and verifying that all nodes have joined successfully.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Node Preparation: Disable multipathd (Required for Longhorn)](#node-preparation-disable-multipathd-required-for-longhorn)
- [Install RKE2 on Control Plane Nodes](#install-rke2-on-control-plane-nodes)
- [Install RKE2 on Worker Nodes](#install-rke2-on-worker-nodes)
- [Configure the Control Plane Node](#configure-the-control-plane-node)
- [Start the RKE2 Server Service](#start-the-rke2-server-service)
- [Retrieve the Control Plane Join Token](#retrieve-the-control-plane-join-token)
- [Configure Worker Nodes](#configure-worker-nodes)
- [Start the RKE2 Agent Service](#start-the-rke2-agent-service)
- [Verify Cluster Nodes](#verify-cluster-nodes)
- [Persist kubectl Configuration](#persist-kubectl-configuration)

---

## Prerequisites

- All commands in this guide must be executed as the `root` user. Switch to root before proceeding:

  ```bash
  sudo -i
  ```

- Ensure you know the **private IP address** of each control plane and worker node.
- Ensure network connectivity between control plane and worker nodes on port `9345` (RKE2 supervisor) and `6443` (Kubernetes API).

---

## Node Preparation: Disable multipathd (Required for Longhorn)

If you plan to install **Longhorn** as the cluster's storage provider, disable the **`multipathd`** service on **all nodes** before installing Longhorn. Failing to do so causes volume mount failures when pods restart or crash, because `multipathd` claims Longhorn's block devices as multipath candidates.

> **Why:** `multipathd` combines multiple storage paths to the same disk/LUN into a single logical device for redundancy and high availability (typical for SAN setups). By default it does **not** discriminate — it claims **every eligible block device**, including the `/dev/longhorn/*` devices Longhorn creates for each volume. Once a Longhorn device is claimed by `multipathd`, the kubelet can no longer format or mount it.

### Symptom

A `MountVolume.MountDevice failed` error appears in kubelet/pod events when a pod using a Longhorn PVC restarts or is rescheduled:

```
MountVolume.MountDevice failed for volume "pvc-275f1d05-1735-4236-bab6-xxxxxx" : rpc error: code = Internal desc = format of disk "/dev/longhorn/pvc-275f1d05-1735-4236-bab6-xxxxxx" failed: type:("ext4") target:("/var/lib/kubelet/plugins/kubernetes.io/csi/driver.longhorn.io/efd3eb006c0c703ceeb49e2778194d04891c7b1ff42ed79837140db4dbf03515/globalmount") options:("defaults") errcode:(exit status 1) output:(mke2fs 1.47.0 (5-Feb-2023)
```

### Check Whether multipathd Is Active and In Use

Run these on **every node** before disabling, to confirm the current state:

```bash
# 1. Confirm multipathd is active and running
systemctl status multipathd

# 2. Check whether it is in use and if device-mapper multipath devices exist
#    (empty output from both commands means multipathd is not actively mapping anything)
multipath -ll
ls /dev/mapper | grep -E '^mpath|^3600'
```

If `multipath -ll` returns entries that correspond to real SAN/iSCSI LUNs your workloads depend on, **do not** simply disable it — jump to [If multipathd Is Required for SAN](#if-multipathd-is-required-for-san) below.

### Disable multipathd

On **every node**, as root:

```bash
# Stop both the socket and the service
systemctl stop multipathd.socket
systemctl stop multipathd.service

# Disable both so they do not restart at boot
systemctl disable multipathd.socket
systemctl disable multipathd.service
```

### Verify multipathd Is Disabled and Inactive

```bash
systemctl status multipathd.socket multipathd.service --no-pager
systemctl status multipathd
```

Expected: both units report `inactive (dead)` and `disabled`.

### If multipathd Is Required for SAN

If a later initiative (e.g. SAN-backed storage) requires `multipathd`, **do not** re-enable it blindly — you must first configure it to blacklist Longhorn's block devices, otherwise the mount failure returns. Per [Longhorn's official troubleshooting guide](https://longhorn.io/kb/troubleshooting-volume-with-multipath/), add a `blacklist` stanza to `/etc/multipath.conf` that excludes Longhorn devices, then restart the service and verify:

```bash
# Example starting point — review against your SAN vendor's guidance before applying
cat >> /etc/multipath.conf <<'EOF'
blacklist {
    devnode "^sd[a-z0-9]+"
}
EOF

systemctl restart multipathd.service
multipath -t   # confirms the active configuration
```

### References

- [Longhorn KB — Troubleshooting: `MountVolume.SetUp failed for volume` due to multipathd on the node](https://longhorn.io/kb/troubleshooting-volume-with-multipath/)
- [Longhorn GitHub #1210 — BUG: `MountVolume.SetUp failed for volume ~ rpc error: code = Internal desc = exit status 1` due to multipathd on the node](https://github.com/longhorn/longhorn/issues/1210)

> If disabling `multipathd` does not fix the `MountDevice failed` error, consult the GitHub issue above — there are a handful of related root causes (stale mount entries, kubelet plugin socket issues) that surface with the same error message.

---

## Install RKE2 on Control Plane Nodes

Run the following commands on **each control plane node** as the `root` user:

```bash
curl -sfL https://get.rke2.io | sudo sh -
systemctl enable rke2-server.service
systemctl status rke2-server.service
```

This installs the RKE2 server binary and enables the `rke2-server` systemd service so it starts automatically on boot. Do **not** start the service yet — it must be configured first (see [Configure the Control Plane Node](#configure-the-control-plane-node)).

---

## Install RKE2 on Worker Nodes

Run the following commands on **each worker node** as the `root` user:

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
systemctl enable rke2-agent.service
systemctl status rke2-agent.service
```

The `INSTALL_RKE2_TYPE="agent"` environment variable instructs the installer to set up an agent (worker) node instead of a server. Do **not** start the service yet — it requires configuration pointing to the control plane (see [Configure Worker Nodes](#configure-worker-nodes)).

---

## Configure the Control Plane Node

On **each control plane node**, create the RKE2 configuration directory and write the cluster configuration file:

```bash
mkdir -p /etc/rancher/rke2/ && sudo sh -c 'cat > /etc/rancher/rke2/config.yaml' <<'EOF'
node-ip: <control-plane-private-ip>
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16

# disable ingress-nginx for traefik
disable:
  - rke2-ingress-nginx

# expose etcd metrics
etcd-expose-metrics: true

# data at rest encryption
secrets-encryption-provider: secretbox

# deny externalIPs
kube-apiserver-arg:
  - "enable-admission-plugins=DenyServiceExternalIPs"
EOF
```

Replace `<control-plane-private-ip>` with the actual private IP address of the control plane node.

**Configuration breakdown:**

| Directive | Purpose |
|-----------|---------|
| `node-ip` | Private IP the node advertises to the cluster. |
| `cluster-cidr` | Pod network CIDR. |
| `service-cidr` | Kubernetes service network CIDR. |
| `disable: rke2-ingress-nginx` | Disables the bundled ingress-nginx controller so Traefik (or another ingress) can be used instead. |
| `etcd-expose-metrics: true` | Exposes etcd metrics for scraping by Prometheus. |
| `secrets-encryption-provider: secretbox` | Enables at-rest encryption of Kubernetes secrets using the `secretbox` provider. |
| `kube-apiserver-arg: enable-admission-plugins=DenyServiceExternalIPs` | Hardens the API server by denying services from using arbitrary external IPs. |

> **Optional — CIS Benchmark hardening:** This base config is functional but not CIS-hardened. RKE2 ships a built-in `profile: cis` mode you can enable on top of this guide — it is **optional** and works for both a **new cluster** (add `profile: cis` to this `config.yaml` before the first start) and an **existing, already-running cluster** (append the profile and do a rolling restart). The profile has prerequisites (a dedicated `etcd` user, required kernel sysctls, and per-namespace NetworkPolicies), all of which are covered step-by-step — for both scenarios — in [rke2-cis-self-assessment-benchmark-config.md](rke2-cis-self-assessment-benchmark-config.md). If you intend to harden a new cluster, read that guide before starting the services below, since it modifies this `config.yaml`.

---

## Start the RKE2 Server Service

Once the config file is in place, start the RKE2 server on the control plane:

```bash
systemctl start rke2-server.service
systemctl status rke2-server.service
```

Wait until the service reports `active (running)` before proceeding. Initial startup may take a few minutes while RKE2 bootstraps etcd, the control plane components, and core add-ons.

---

## Retrieve the Control Plane Join Token

Worker nodes need the cluster token to authenticate with the control plane. On the control plane node, retrieve it with:

```bash
cat /var/lib/rancher/rke2/server/node-token
```

Copy the token — you will use it in the worker node configuration below.

---

## Configure Worker Nodes

Switch to the `root` user on each worker node and create the agent configuration file:

```bash
mkdir -p /etc/rancher/rke2/ && sh -c 'cat > /etc/rancher/rke2/config.yaml' <<'EOF'
server: https://<control-plane-private-ip>:9345
token: <control-plane-token>
node-ip: <worker-node-private-ip>

# Kubelet Strong Cryptographic Ciphers
kubelet-arg:
  - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305"
  - "pod-max-pids=400"
  - "seccomp-default=true"
EOF
```

Replace the placeholders:

- `<control-plane-private-ip>` — private IP of the control plane node.
- `<control-plane-token>` — token retrieved in the [previous step](#retrieve-the-control-plane-join-token).
- `<worker-node-private-ip>` — private IP of the worker node being configured.

**Configuration breakdown:**

| Directive | Purpose |
|-----------|---------|
| `server` | URL of the RKE2 control plane supervisor (port `9345`). |
| `token` | Cluster join token obtained from the control plane. |
| `node-ip` | Private IP the worker advertises to the cluster. |
| `kubelet-arg: tls-cipher-suites=...` | Restricts kubelet to strong cryptographic cipher suites. |
| `kubelet-arg: pod-max-pids=400` | Limits the maximum number of PIDs per pod to mitigate fork bombs. |
| `kubelet-arg: seccomp-default=true` | Enables the default seccomp profile for all workloads. |

---

## Start the RKE2 Agent Service

Once the worker config file is in place, start the agent on **each worker node**:

```bash
systemctl start rke2-agent.service
systemctl status rke2-agent.service
```

The worker will contact the control plane at `https://<control-plane-private-ip>:9345`, authenticate using the token, and join the cluster.

---

## Verify Cluster Nodes

On the **main control plane node**, set up `kubectl` and verify that all nodes have joined:

```bash
ln -s /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/kubectl
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
kubectl get nodes
kubectl get pods -A
```

All control plane and worker nodes should be listed with status `Ready`, and core system pods (CoreDNS, canal/flannel, metrics-server, etc.) should be `Running`.

---

## Persist kubectl Configuration

To ensure the `KUBECONFIG` environment variable is available in every future root shell, append it to root's `.bashrc`:

```bash
echo 'export KUBECONFIG=/etc/rancher/rke2/rke2.yaml' >> /root/.bashrc
source /root/.bashrc
```

Your RKE2 cluster is now installed and ready for workload deployment.
