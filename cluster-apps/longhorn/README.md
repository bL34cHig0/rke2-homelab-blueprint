# Longhorn Distributed Block Storage on RKE2

## Table of Contents

- [Overview](#overview)
- [Files](#files)
- [Prerequisites](#prerequisites)
  - [Install node prerequisites](#install-node-prerequisites)
  - [multipathd](#multipathd)
- [Install Longhorn via the Rancher UI](#install-longhorn-via-the-rancher-ui)
  - [Step 1: Open the App Marketplace](#step-1-open-the-app-marketplace)
  - [Step 2: Select the Longhorn Chart](#step-2-select-the-longhorn-chart)
  - [Step 3: Review Cluster Readiness](#step-3-review-cluster-readiness)
  - [Step 4: Configure the Chart](#step-4-configure-the-chart)
  - [Step 5: Install and Watch the Rollout](#step-5-install-and-watch-the-rollout)
  - [Step 6: Verify](#step-6-verify)
  - [Step 7: Access the Longhorn UI](#step-7-access-the-longhorn-ui)
- [Encrypted StorageClass](#encrypted-storageclass)
  - [How Longhorn Encryption Works](#how-longhorn-encryption-works)
  - [Global vs Per-Volume Keys](#global-vs-per-volume-keys)
  - [Step 1: Create the Encryption Secret](#step-1-create-the-encryption-secret)
  - [Step 2: Apply the Encrypted StorageClass](#step-2-apply-the-encrypted-storageclass)
  - [Step 3: Verify Encryption End-to-End](#step-3-verify-encryption-end-to-end)
  - [Per-Volume Keys (Optional)](#per-volume-keys-optional)
- [Operations](#operations)
  - [Set a Default StorageClass](#set-a-default-storageclass)
  - [Upgrades](#upgrades)
- [Troubleshooting](#troubleshooting)

## Overview

[Longhorn](https://longhorn.io/) is a lightweight, cloud-native distributed block storage system for Kubernetes. It provisions replicated `PersistentVolume` resources backed by the `driver.longhorn.io` CSI driver and is the storage layer every PVC in this repo binds to (see the `monitoring` and `wordpress` storage classes).

This folder documents installing Longhorn through the **Rancher Apps & Marketplace UI** (rather than Helm directly) and creating a **LUKS-encrypted StorageClass** for workloads that need encryption at rest.

**Helm Chart**: [longhorn/longhorn](https://github.com/longhorn/longhorn/tree/master/chart) — surfaced in Rancher as a built-in marketplace chart.

## Files

| File | Description |
|------|-------------|
| `encryption-secret.yaml` | **Template** showing the shape of the LUKS Secret (`longhorn-crypto`). Created imperatively, not applied directly — see [Step 1](#step-1-create-the-encryption-secret) |
| `encrypted-storage-class.yaml` | `StorageClass` (name is a `<your-storage-class>` placeholder) with `encrypted: "true"` wired to the `longhorn-crypto` Secret for all CSI secret stages |
| `ingress.yaml` | Traefik `IngressRoute` exposing the `longhorn-frontend` UI at `longhorn.<your-domain>` behind the basic-auth + default-headers middlewares (optional — only if exposing the UI publicly) |
| `middleware.yaml` | Traefik `Middleware` resources — `longhorn-basicauth` (references the `longhorn-auth` Secret) and `default-headers` (security headers) |
| `secret-longhorn-auth.yaml` | **Template** showing the shape of the htpasswd Secret (`longhorn-auth`). Created imperatively, not applied directly |
| `longhorn-certificate.yaml` | cert-manager `Certificate` that writes the wildcard `default-cert-production-tls` secret into `longhorn-system` (the IngressRoute's TLS secret must live in this namespace) |

## Prerequisites

- An RKE2 cluster managed by **Rancher** (see [`../rancher/`](../rancher/)), provisioned per the [`../../infrastructure/`](../../infrastructure/) docs.
- `multipathd` disabled on every node — **already covered** during node setup in [rke2-installation.md → Disable multipathd](../../infrastructure/rke2-installation.md). Confirm it's done before installing Longhorn; the [troubleshooting table](#troubleshooting) explains the symptom if it wasn't.
- The `open-iscsi` package installed and `iscsid` running on **every** node (not handled by the infrastructure prerequisites — install it below).
- The `nfs-common` package on every node if you plan to use RWX (ReadWriteMany) volumes or NFS-backed backups.
- A dedicated disk or sufficient free space under `/var/lib/longhorn` on each node that will host replicas.
- The `dm_crypt` kernel module loaded on every node for the encrypted StorageClass.

### Install node prerequisites

These nodes run **Ubuntu 24.04** (see [rke2-ubuntu-prerequisites.md](../../infrastructure/rke2-ubuntu-prerequisites.md)). The RKE2 node setup installs the base packages but **not** `open-iscsi`, `nfs-common`, or the `dm_crypt` module — install them here. Run on **every** node (as root, or prefix with `sudo`).

**Check what's already present first:**

```bash
systemctl status iscsid          # iSCSI daemon (required by Longhorn)
lsmod | grep dm_crypt            # encryption module (required for the encrypted class)
```

**Install `open-iscsi` (and `nfs-common` for RWX/NFS backups):**

```bash
apt-get update
apt-get install -y open-iscsi nfs-common
systemctl enable --now iscsid
```

**Load the `dm_crypt` module** (needed only for the encrypted StorageClass) and make it persist across reboots:

```bash
modprobe dm_crypt
echo dm_crypt > /etc/modules-load.d/dm_crypt.conf   # auto-load on boot
```

**Verify after installing:**

```bash
systemctl status iscsid           # active (running)
lsmod | grep dm_crypt             # module listed
```

> Note: Longhorn previously shipped `kubectl`-applied DaemonSets to install `open-iscsi`/`nfs-client` cluster-wide, but these were removed in v1.12.0. Install the packages per node as above, or see the [Longhorn installation docs](https://longhorn.io/docs/1.12.0/deploy/install/) for the current installation-requirements guidance.

### multipathd

`multipathd` claims Longhorn's block devices and causes mount failures (`MountVolume.MountDevice failed ... format of disk /dev/longhorn/pvc-... failed`). Disabling it on all nodes is part of node setup and is documented in full — including the SAN-blacklist path if you need multipathd later — in [infrastructure/rke2-installation.md → Disable multipathd](../../infrastructure/rke2-installation.md). Confirm it before installing Longhorn:

```bash
systemctl status multipathd       # should be inactive (dead) / disabled
```

## Install Longhorn via the Rancher UI

> Note: This guide covers the Rancher Apps & Marketplace UI path. Longhorn can also be installed via Helm, a `kubectl` manifest, or the Rancher Fleet/GitOps flow. For the full list of installation methods and their requirements, refer to the official [Longhorn installation docs](https://longhorn.io/docs/1.12.0/deploy/install/).

### Step 1: Open the App Marketplace

1. Log in to Rancher and open the **local** (or target) cluster from the cluster list.
2. In the left navigation, go to **Apps → Charts**.

### Step 2: Select the Longhorn Chart

1. In the chart search box, type `Longhorn`.
2. Click the **Longhorn** card (published by SUSE/Rancher — it carries the official Longhorn logo).
3. On the chart page, use the **version dropdown** (top right) to pick a chart version. Cross-reference the [Longhorn releases](https://github.com/longhorn/longhorn/releases) and the [support matrix](https://longhorn.io/docs/latest/deploy/important-notes/) to confirm it supports your RKE2 Kubernetes version. Prefer the latest **stable** (non-`-rc`, non-`-dev`) version.
4. Click **Install**.

### Step 3: Review Cluster Readiness

Rancher runs a preflight check and Longhorn ships an **environment check**. Before continuing, confirm on every node:

- `iscsid` is running.
- `multipathd` is disabled (see [above](#multipathd)).
- For encryption, `dm_crypt` is loadable.

You can also run Longhorn's official environment check script ahead of time:

```bash
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/master/scripts/environment_check.sh | bash
```

### Step 4: Configure the Chart

The install wizard opens with two tabs — a guided form and **Edit YAML**. The defaults are sensible for a starter cluster; adjust these for this homelab:

| Setting | Recommendation | Why |
|---|---|---|
| **Default Replica Count** | `1` for a single storage node, `2`–`3` for multi-node | Matches the `numberOfReplicas` used by the storage classes in this repo. More replicas = more resilience but more disk. |
| **Default Data Path** | `/var/lib/longhorn` (default) | Change only if replica data must live on a dedicated mount. |
| **Default Longhorn Static StorageClass Name** | leave default (`longhorn`) | The unencrypted default class Longhorn creates automatically. |
| **Storage Over Provisioning Percentage** | lower (e.g. `100`) for homelab | Prevents scheduling more volume capacity than physically exists. |
| **Pod Security Policy / namespace** | namespace `longhorn-system` (fixed) | All Longhorn components and the encryption Secret live here. |

> Note: Do **not** enable Longhorn's own ingress in the chart. This cluster fronts UIs with Traefik — expose the Longhorn UI via a Traefik `IngressRoute` later if needed (see [Step 7](#step-7-access-the-longhorn-ui)).

### Step 5: Install and Watch the Rollout

Click **Install**. Rancher opens a Helm operation log pane. Installation pulls several images and can take a few minutes. You can also follow it from a shell:

```bash
kubectl -n longhorn-system rollout status deploy/longhorn-driver-deployer
kubectl get pods -n longhorn-system
```

### Step 6: Verify

```bash
# all pods Running/Completed
kubectl get pods -n longhorn-system

# the CSI driver is registered
kubectl get csidrivers | grep longhorn

# the default StorageClass exists
kubectl get storageclass

# every node reports schedulable storage
kubectl get nodes.longhorn.io -n longhorn-system
```

### Step 7: Access the Longhorn UI

Once installed, the quickest way in is **through Rancher itself**: in the Rancher dashboard, open the cluster and look in the left-hand navigation panel — a **Longhorn** entry appears under the cluster's **Storage / Monitoring** tools. Clicking it embeds the Longhorn UI inside Rancher, proxied through the authenticated Rancher session, so no separate exposure is needed.

For access outside Rancher, the chart also deploys a `longhorn-frontend` Service in `longhorn-system`. For a quick look without exposing it publicly:

```bash
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80
# browse to http://localhost:8080
```

#### Exposing the UI publicly (optional)

> **Default for this blueprint: don't.** Access Longhorn through the Rancher dashboard (above). Rancher already authenticates the session and proxies the UI, so there's nothing extra to secure — leaving Longhorn unexposed keeps the attack surface minimal. Only add a public route if you specifically need to reach Longhorn without going through Rancher.

If you do expose it, front it with a Traefik `IngressRoute` and a **basic-auth** middleware — the Longhorn UI has **no built-in authentication**, so it must never be exposed without one. The Longhorn docs show this with a plain Kubernetes [`Ingress`](https://longhorn.io/docs/1.12.0/deploy/accessing-the-ui/longhorn-ingress-traefik/), but this blueprint uses Traefik `IngressRoute` CRDs throughout, so the manifests in this folder provide the IngressRoute equivalent.

The four manifests — `longhorn-certificate.yaml`, `secret-longhorn-auth.yaml` (template), `middleware.yaml`, and `ingress.yaml` — mirror the [`../traefik/dashboard/`](../traefik/dashboard/) pattern. Apply them in this order:

```bash
# 1. Issue the wildcard TLS cert into longhorn-system (the IngressRoute's TLS
#    secret must live in the same namespace as the route).
kubectl apply -f longhorn-certificate.yaml
kubectl get certificate -n longhorn-system   # wait for READY=True

# 2. Create the htpasswd basic-auth Secret imperatively (never commit credentials).
#    -B = bcrypt; copy the full "user:$2y$..." line into the secret.
htpasswd -nbB <username> <strong-password>
kubectl create secret generic longhorn-auth -n longhorn-system \
  --from-literal=users='<htpasswd-output>'

# 3. Apply the middlewares (basic-auth + default-headers) and the route.
kubectl apply -f middleware.yaml
kubectl apply -f ingress.yaml

# 4. Verify
kubectl get ingressroute,middleware -n longhorn-system
```

Then point `longhorn.<your-domain>` at the ingress node (or rely on the existing wildcard DNS record) and browse to `https://longhorn.<your-domain>` — Traefik will prompt for the basic-auth credentials. Replace the `<your-domain>` placeholders in `ingress.yaml` and `longhorn-certificate.yaml` first. `secret-longhorn-auth.yaml` is a **template** showing the Secret's shape — don't apply it directly; create the Secret with the commands above.

## Encrypted StorageClass

### How Longhorn Encryption Works

Longhorn encrypts volumes at rest with **LUKS** (`dm_crypt`). When a PVC uses an encryption-enabled StorageClass, the CSI driver:

1. Reads the passphrase from a Kubernetes Secret (referenced by the StorageClass via the standard `csi.storage.k8s.io/*-secret-*` parameters).
2. Creates a LUKS-formatted block device on the node (`/dev/mapper/...`) at attach time.
3. Mounts the decrypted device into the pod. Data on the underlying replica disks and in backups is ciphertext.

The passphrase lives only in the Secret; if the Secret is deleted, existing encrypted volumes can no longer be unlocked, so back it up securely.

### Global vs Per-Volume Keys

| Model | Key scope | Setup |
|---|---|---|
| **Global** (this folder) | One passphrase shared by every volume from the class | One Secret + one StorageClass. Simplest; used below. |
| **Per-volume** | A distinct Secret per PVC namespace/name | StorageClass references templated secret names (`${pvc.namespace}`, `${pvc.name}`). Stronger blast-radius isolation. See [below](#per-volume-keys-optional). |

### Step 1: Create the Encryption Secret

> `encryption-secret.yaml` in this directory is a **template** showing the Secret's shape. Don't put a real passphrase in it or apply it directly — create the Secret imperatively so the passphrase never lands in the repo. Generate a strong passphrase with a password manager and store it there.

```bash
# generate a strong random passphrase (and copy it into your password manager)
CRYPTO_KEY=$(openssl rand -base64 32)

# create the longhorn-system namespace if Longhorn isn't installed yet
kubectl create namespace longhorn-system --dry-run=client -o yaml | kubectl apply -f -

# create the Secret imperatively
kubectl create secret generic longhorn-crypto -n longhorn-system \
  --from-literal=CRYPTO_KEY_VALUE="$CRYPTO_KEY" \
  --from-literal=CRYPTO_KEY_PROVIDER="secret" \
  --from-literal=CRYPTO_KEY_CIPHER="aes-xts-plain64" \
  --from-literal=CRYPTO_KEY_HASH="sha256" \
  --from-literal=CRYPTO_KEY_SIZE="256" \
  --from-literal=CRYPTO_PBKDF="argon2i"

# clear the variable from the shell
unset CRYPTO_KEY

# verify the keys exist (values stay base64-encoded / hidden)
kubectl describe secret longhorn-crypto -n longhorn-system
```

> Caveat: Make sure shell history is not recording the passphrase. The [rancher README](../rancher/README.md#prerequisites) shows the `HISTIGNORE`/`HISTTIMEFORMAT` setup that excludes `openssl*` and `*secret*` commands.

### Step 2: Apply the Encrypted StorageClass

Replace the `<your-storage-class>` placeholder in `encrypted-storage-class.yaml` with a name that fits your convention (the other apps in this repo each use their own, e.g. `monitoring`), then apply:

```bash
kubectl apply -f encrypted-storage-class.yaml

# confirm it exists and points at the encryption Secret
kubectl get storageclass <your-storage-class>
kubectl describe storageclass <your-storage-class>
```

The class in `encrypted-storage-class.yaml` mirrors the conventions of the other storage classes in this repo (`reclaimPolicy: Retain`, `numberOfReplicas: "1"`, `fsType: ext4`) and adds `encrypted: "true"` plus the three CSI secret stages (provisioner, node-stage, node-publish) all pointing at `longhorn-crypto`.

### Step 3: Verify Encryption End-to-End

Provision a test PVC against the class and confirm the device is LUKS-encrypted:

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: crypto-test
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: <your-storage-class>
  resources:
    requests:
      storage: 1Gi
EOF

# wait for it to bind
kubectl get pvc crypto-test -w
```

Once bound and attached (mount it in a throwaway pod, or check the Longhorn UI), confirm the backing device is a crypt device on the node hosting the replica:

```bash
# on the node, the decrypted mapper device shows type "crypt"
lsblk -o NAME,TYPE,MOUNTPOINT | grep crypt

# Longhorn marks the volume as encrypted
kubectl get volumes.longhorn.io -n longhorn-system
```

In the Longhorn UI, the volume's detail page shows **Encrypted: Yes**. Clean up with `kubectl delete pvc crypto-test`.

### Per-Volume Keys (Optional)

For per-PVC keys instead of one global passphrase, create a StorageClass whose secret parameters use Longhorn's template variables, then create one Secret per PVC (named `<pvc-name>` in namespace `<pvc-namespace>`):

```yaml
parameters:
  encrypted: "true"
  csi.storage.k8s.io/provisioner-secret-name: ${pvc.name}
  csi.storage.k8s.io/provisioner-secret-namespace: ${pvc.namespace}
  csi.storage.k8s.io/node-publish-secret-name: ${pvc.name}
  csi.storage.k8s.io/node-publish-secret-namespace: ${pvc.namespace}
  csi.storage.k8s.io/node-stage-secret-name: ${pvc.name}
  csi.storage.k8s.io/node-stage-secret-namespace: ${pvc.namespace}
```

This isolates blast radius (compromising one key exposes one volume) at the cost of managing a Secret per workload. See the Longhorn [encryption docs](https://longhorn.io/docs/1.12.0/advanced-resources/security/volume-encryption/) for details.

## Operations

### Set a Default StorageClass

Only one StorageClass should carry the default annotation. To make the encrypted class the cluster default (so PVCs without `storageClassName` get encrypted volumes):

```bash
# remove default from the built-in longhorn class
kubectl patch storageclass longhorn \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# set the encrypted class as default
kubectl patch storageclass <your-storage-class> \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# confirm exactly one (default) marker
kubectl get storageclass
```

### Upgrades

Upgrade Longhorn from the same Rancher screen used to install it:

1. **Apps → Installed Apps → longhorn → ⋮ → Edit/Upgrade**.
2. Pick the new chart version (only upgrade one minor version at a time — Longhorn does not support skipping minors).
3. Read the target version's [upgrade notes](https://longhorn.io/docs/latest/deploy/upgrade/) first; some releases require all volumes to be healthy and detached.

```bash
# watch the upgrade
kubectl -n longhorn-system get pods -w

# confirm the new version
kubectl -n longhorn-system get settings.longhorn.io current-longhorn-version
```

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `MountVolume.MountDevice failed ... format of disk /dev/longhorn/pvc-... failed` | `multipathd` is claiming the device. [Disable it](#multipathd) on all nodes. |
| Encrypted PVC stuck `Pending` / volume won't attach | `dm_crypt` not loaded on the node (`modprobe dm_crypt`), or the `longhorn-crypto` Secret is missing/misnamed in `longhorn-system`. |
| Volume attaches but data is unreadable after Secret change | The LUKS passphrase cannot be changed in place by editing the Secret — the original passphrase is needed to unlock existing volumes. Restore the original `CRYPTO_KEY_VALUE`. |
| Pods can't schedule due to storage | Node marked unschedulable in Longhorn, disk full, or over-provisioning limit hit. Check the Longhorn UI **Node** page. |
| Longhorn UI shows no schedulable storage | `iscsid` not running, or no node has a writable `/var/lib/longhorn`. |

```bash
# inspect Longhorn manager logs
kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200

# check node/disk scheduling status
kubectl get nodes.longhorn.io -n longhorn-system -o wide

# CSI plugin logs (attach/mount/encryption errors surface here)
kubectl -n longhorn-system logs -l app=longhorn-csi-plugin -c longhorn-csi-plugin --tail=200
```

For deeper issues, see the Longhorn [troubleshooting guide](https://longhorn.io/docs/1.12.0/troubleshoot/troubleshooting/) and [volume encryption docs](https://longhorn.io/docs/1.12.0/advanced-resources/security/volume-encryption/).
