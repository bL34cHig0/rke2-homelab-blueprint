# RKE2 CIS Self-Assessment Benchmark Configuration

This guide documents how to enable RKE2's built-in `profile: cis` hardening mode. It is written for **two scenarios**:

- **Scenario A — New cluster:** Applying CIS hardening as part of the initial RKE2 install, before any workloads are deployed.
- **Scenario B — Existing cluster:** Enabling CIS on a running cluster that already has Rancher, Longhorn, MetalLB, Traefik, cert-manager, Fleet, NeuVector, kube-prometheus-stack, and custom workloads deployed.

Each step below flags which scenario it applies to. The core work (etcd user, sysctls, profile flag) is identical in both; the NetworkPolicy rollout (Step 3) differs significantly.

For the base install itself, see [rke2-installation.md](rke2-installation.md). This guide is the security hardening layer on top.

---

## Table of Contents

- [Confirm Your RKE2 Version](#confirm-your-rke2-version)
- [What the CIS Profile Does](#what-the-cis-profile-does)
- [Scenario at a Glance](#scenario-at-a-glance)
- [Pre-Flight Checklist](#pre-flight-checklist)
- [Step 1: Create the etcd System User and Group](#step-1-create-the-etcd-system-user-and-group)
  - [Why a Dedicated etcd User](#why-a-dedicated-etcd-user)
  - [Verify Current State First](#verify-current-state-first)
  - [Create the User](#create-the-user)
  - [Verify the User Was Created Correctly](#verify-the-user-was-created-correctly)
- [Step 2: Apply Required Kernel Parameters](#step-2-apply-required-kernel-parameters)
  - [Why These sysctls Are Required](#why-these-sysctls-are-required)
  - [Verify Current sysctl State First](#verify-current-sysctl-state-first)
  - [Apply the sysctls](#apply-the-sysctls)
  - [Verify the sysctls Took Effect](#verify-the-sysctls-took-effect)
- [Step 3: Satisfy the NetworkPolicy Requirement (CIS 5.3.2)](#step-3-satisfy-the-networkpolicy-requirement-cis-532)
  - [What CIS 5.3.2 Requires](#what-cis-532-requires)
  - [Scenario A — New Cluster NetworkPolicy Strategy](#scenario-a--new-cluster-networkpolicy-strategy)
  - [Scenario B — Existing Cluster NetworkPolicy Strategy](#scenario-b--existing-cluster-networkpolicy-strategy)
    - [Why a Blanket Default-Deny Will Break the Cluster](#why-a-blanket-default-deny-will-break-the-cluster)
    - [Vendor Guidance Reality Check](#vendor-guidance-reality-check)
    - [The Pragmatic Approach: allow-all Placeholders](#the-pragmatic-approach-allow-all-placeholders)
    - [Audit Namespaces Missing Policies](#audit-namespaces-missing-policies)
    - [Verify Rancher Bookkeeping Namespaces Have No Pods](#verify-rancher-bookkeeping-namespaces-have-no-pods)
    - [Canary a Single Rancher-Managed Namespace First](#canary-a-single-rancher-managed-namespace-first)
    - [Dry-Run the Full Rollout](#dry-run-the-full-rollout)
    - [Apply allow-all to All Remaining Namespaces](#apply-allow-all-to-all-remaining-namespaces)
    - [Final NetworkPolicy Verification](#final-networkpolicy-verification)
- [Step 3b: Disable Default ServiceAccount Token Automount (CIS 5.1.5)](#step-3b-disable-default-serviceaccount-token-automount-cis-515)
  - [Why This Matters](#why-this-matters)
  - [Verify Current State First](#verify-current-state-first-1)
  - [Patch All default ServiceAccounts](#patch-all-default-serviceaccounts)
  - [Verify the Patch Took Effect](#verify-the-patch-took-effect)
  - [Important Caveat](#important-caveat)
- [Step 4: Enable the CIS Profile in RKE2](#step-4-enable-the-cis-profile-in-rke2)
  - [Verify config.yaml Current State First](#verify-configyaml-current-state-first)
  - [Scenario A — New Cluster: Author config.yaml With profile: cis](#scenario-a--new-cluster-author-configyaml-with-profile-cis)
  - [Scenario B — Existing Cluster: Append profile: cis and Restart](#scenario-b--existing-cluster-append-profile-cis-and-restart)
- [Step 5: Verify CIS Hardening Took Effect](#step-5-verify-cis-hardening-took-effect)
- [Step 6: Re-Run the RKE2 CIS Self-Assessment](#step-6-re-run-the-rke2-cis-self-assessment)
- [Future Hardening (Technical Debt)](#future-hardening-technical-debt)
- [Troubleshooting](#troubleshooting)

---

## Confirm Your RKE2 Version

Before anything else, confirm which RKE2 version you are running (existing cluster) or planning to install (new cluster) — the correct `profile` value depends on it.

**Existing cluster — check on each node:**

```bash
rke2 --version
```

Example output:

```
rke2 version v1.34.3+rke2r1 (<alpha-numeric-string>)
go version go1.24.11 X:boringcrypto
```

**New cluster — check what the installer will fetch** (before running the install script):

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=<version> sh -s -- --help 2>/dev/null || \
  curl -s https://update.rke2.io/v1-release/channels | grep -oE '"latest":"v[^"]+"'
```

Map your version to the correct profile string:

| RKE2 version | Profile value |
|--------------|---------------|
| v1.24 and earlier | `cis-1.6` |
| v1.25 – v1.28 | `cis-1.23` |
| v1.29+ (including v1.34.x) | `cis` *(tracks latest supported benchmark)* |

> **Important:** The string `cis-1.11` is **not** a valid profile value. "1.11" refers to the RKE2 self-assessment document version, not the benchmark version the profile flag accepts. For v1.34.x, use `profile: cis` — RKE2 internally resolves this to the latest benchmark it supports. The mapping of those controls to CIS 1.11 numbering is maintained at <https://docs.rke2.io/security/cis_self_assessment111>.

**Existing cluster only** — also confirm the cluster is currently healthy. Hardening an already-broken cluster is not safe:

```bash
kubectl get nodes
kubectl get pods -A | grep -vE 'Running|Completed' || echo "All pods Running/Completed"
```

---

## What the CIS Profile Does

Setting `profile: cis` in `/etc/rancher/rke2/config.yaml` flips RKE2 from its default (functional, non-hardened) configuration into CIS-compliant mode. When the profile is active, RKE2 enforces the following automatically:

1. **etcd runs as a dedicated unprivileged user**, not root.
   - Checks `/etc/passwd` and `/etc/group` for an `etcd` user/group — refuses to start if missing.
   - Chowns `/var/lib/rancher/rke2/server/db/etcd` to `etcd:etcd`.
   - Injects `runAsUser` / `runAsGroup` into the etcd static pod's `SecurityContext`.
2. **Kubelet sets `protect-kernel-defaults=true`** — kubelet exits if required kernel parameters are unset or wrong.
3. **Admission plugins and API server hardening** — including `NodeRestriction`, `PodSecurity` (restricted profile by default on non-system namespaces), and other CIS-mandated plugins.
4. **Default NetworkPolicies** are created by RKE2 in `kube-system`, `kube-public`, and `default`.
5. **Audit logging and other kube-apiserver hardening flags** are set to CIS-required values.

Controls in the **5.x Policies** range of the benchmark (`NetworkPolicy` presence on user namespaces, PodSecurity on custom workloads, RBAC hygiene, etc.) are the operator's responsibility — RKE2 cannot enforce them on workloads it does not manage.

---

## Scenario at a Glance

| Step | Scenario A — New cluster | Scenario B — Existing cluster |
|------|--------------------------|-------------------------------|
| 1. Create etcd user | Before first `systemctl start rke2-server` | Before restarting `rke2-server` |
| 2. Apply sysctls | Before first `systemctl start` on each node | Before restarting each node |
| 3. NetworkPolicies | None needed pre-install; apply to user namespaces **as you deploy workloads** | Audit existing namespaces, apply `allow-all` placeholders before the profile flip |
| 3b. Default SA token automount | Run after each new namespace is created (or wrap into a namespace-bootstrap template) | Patch all existing namespaces' `default` SA before the profile flip |
| 4. Enable `profile: cis` + `service-account-extend-token-expiration=false` | Include both in initial `config.yaml` | Append both to existing `config.yaml`, then rolling restart |
| 5. Verify | Same verification commands apply | Same verification commands apply |

---

## Pre-Flight Checklist

**Both scenarios:**

- [ ] All control plane and worker nodes will run the same RKE2 version.
- [ ] You have root access to every node.
- [ ] You know the private IP of every node.

**Scenario A (new cluster) additionally:**

- [ ] You have followed (or will follow) the base steps in [rke2-installation.md](rke2-installation.md) — this guide modifies that flow to build in CIS hardening from the start.
- [ ] No `rke2-server` or `rke2-agent` service has been started yet (or if the install script has run, the service is still stopped).

**Scenario B (existing cluster) additionally:**

- [ ] Verified via `rke2 --version` on each node.
- [ ] Cluster is currently healthy (`kubectl get nodes` shows all Ready; no crashlooping system pods).
- [ ] A recent etcd snapshot has been taken — etcd data directory ownership changes are safe but a snapshot is cheap insurance.
- [ ] You have a maintenance window or the cluster can tolerate a rolling restart of `rke2-server` and `rke2-agent`.
- [ ] You have identified all namespaces that currently lack NetworkPolicies (see [Step 3 / Audit](#audit-namespaces-missing-policies)).

---

## Step 1: Create the etcd System User and Group

**Applies to: Both scenarios. Run on every control plane node.**

### Why a Dedicated etcd User

The CIS Benchmark requires that etcd — the key-value store holding **all cluster state and secrets** — be owned and run by a **dedicated, unprivileged `etcd` user/group** rather than `root`. This limits blast radius: if the etcd process is compromised, the attacker gets an unprivileged account, not root, and only the etcd data directory (mode `0700`, owned by `etcd:etcd`) is accessible.

RKE2's enforcement of this (when `profile: cis` is set) depends on the user being present in the **flat-file databases** (`/etc/passwd`, `/etc/group`). RKE2's Go runtime uses `os/user` which does **not** consult NSS modules such as SSSD, LDAP, or `systemd-userdb`. A user visible via `getent` but not via a direct grep of `/etc/passwd` will be rejected.

### Verify Current State First

On each control plane node, check whether the user already exists:

```bash
id etcd 2>/dev/null || echo "etcd user does not exist"
grep '^etcd:' /etc/passwd /etc/group 2>/dev/null || echo "etcd not in flat-file databases"
```

**Existing cluster only** — also check the etcd data directory ownership:

```bash
ls -ld /var/lib/rancher/rke2/server/db/etcd 2>/dev/null || echo "etcd data dir does not exist yet"
```

- If `id etcd` returns a user **and** the `grep` shows entries in both files, skip to [Step 2](#step-2-apply-required-kernel-parameters).
- If the user does not exist in the flat files, proceed to create it.
- If the user exists but the data directory is still owned by `root:root` (existing cluster), that is expected — RKE2 will chown it on the next restart once `profile: cis` is active.

### Create the User

On each control plane node, as root:

```bash
sudo useradd -r -c "etcd user" -s /sbin/nologin -M etcd -U
```

Flag breakdown:

| Flag | Purpose |
|------|---------|
| `-r` | Create a system account (UID/GID in the system range). |
| `-c "etcd user"` | Human-readable GECOS comment. |
| `-s /sbin/nologin` | Prevent interactive login. |
| `-M` | Do not create a home directory. |
| `-U` | Create a group with the same name as the user. Required because some distros (e.g. certain RHEL/SUSE variants) do not auto-create a matching group. |

### Verify the User Was Created Correctly

```bash
grep '^etcd:' /etc/passwd /etc/group
```

Expected output (UID/GID values may differ):

```
/etc/passwd:etcd:x:999:988:etcd user:/home/etcd:/sbin/nologin
/etc/group:etcd:x:988:
```

> If `getent passwd etcd` returns a user but `grep '^etcd:' /etc/passwd` does not, the account is coming from NSS/LDAP/SSSD and RKE2 will reject it. You must create a local account.

Also confirm the shell is non-interactive and the account is system-scoped:

```bash
getent passwd etcd
id etcd
```

The shell column must be `/sbin/nologin` (or `/usr/sbin/nologin`).

---

## Step 2: Apply Required Kernel Parameters

**Applies to: Both scenarios. Run on every node (control plane and worker).**

### Why These sysctls Are Required

When `profile: cis` is active, RKE2 automatically sets the kubelet flag `protect-kernel-defaults=true`. This causes the kubelet to **exit** if any of the kernel parameters it depends on are unset or differ from its defaults. RKE2 also runs the same pre-flight check itself and names the violating parameter — a convenience to speed up diagnosis.

Setting `protect-kernel-defaults: false` explicitly while `profile: cis` is active will cause RKE2 to exit with an error. The parameters must actually be correct.

### Verify Current sysctl State First

On every node, check current values before making changes:

```bash
sysctl vm.panic_on_oom vm.overcommit_memory kernel.panic kernel.panic_on_oops kernel.keys.root_maxbytes 2>&1
```

Required values:

| Parameter | Required value |
|-----------|----------------|
| `vm.panic_on_oom` | `0` |
| `vm.overcommit_memory` | `1` |
| `kernel.panic` | `10` |
| `kernel.panic_on_oops` | `1` |
| `kernel.keys.root_maxbytes` | `25000000` |

Also check for pre-existing conflicting entries in other sysctl files (any of these would need to be reconciled to avoid surprises):

```bash
grep -rE 'vm\.panic_on_oom|vm\.overcommit_memory|kernel\.panic|kernel\.panic_on_oops|kernel\.keys\.root_maxbytes' \
  /etc/sysctl.conf /etc/sysctl.d/ 2>/dev/null
```

### Apply the sysctls

On every node (control plane **and** workers — the kubelet runs on both):

```bash
cat > /etc/sysctl.d/90-kubelet.conf <<EOF
vm.panic_on_oom=0
vm.overcommit_memory=1
kernel.panic=10
kernel.panic_on_oops=1
kernel.keys.root_maxbytes=25000000
EOF

sysctl -p /etc/sysctl.d/90-kubelet.conf
```

### Verify the sysctls Took Effect

```bash
sysctl vm.panic_on_oom vm.overcommit_memory kernel.panic kernel.panic_on_oops kernel.keys.root_maxbytes
```

Expected:

```
vm.panic_on_oom = 0
vm.overcommit_memory = 1
kernel.panic = 10
kernel.panic_on_oops = 1
kernel.keys.root_maxbytes = 25000000
```

If any value does not match, check for a later-loaded file under `/etc/sysctl.d/` overriding your config (`ls /etc/sysctl.d/` — files are loaded in lexical order; `90-kubelet.conf` is beaten by anything `91-*` or later).

---

## Step 3: Satisfy the NetworkPolicy Requirement (CIS 5.3.2)

### What CIS 5.3.2 Requires

> *Ensure that all Namespaces have NetworkPolicies defined.*

The control is satisfied by the **presence** of a `NetworkPolicy` in each namespace. The audit does not inspect the policy's contents — an `allow-all` policy passes the control identically to a tight default-deny.

RKE2 auto-creates a default `NetworkPolicy` in `kube-system`, `kube-public`, and `default` when the CIS profile is active. All other namespaces are the operator's responsibility.

---

### Scenario A — New Cluster NetworkPolicy Strategy

On a new cluster there are no workloads yet, so Step 3 is essentially **a policy to adopt going forward** rather than a one-shot rollout:

1. **Skip the bulk rollout entirely.** There are no existing user namespaces to audit.
2. **As you create each new namespace, ship a NetworkPolicy with it.** Every Helm chart deployment, every kustomize overlay, every raw manifest apply should include a NetworkPolicy in the same change.
3. **Start tight, loosen as needed.** Because you know the workload's traffic pattern at deploy time, author a tailored `default-deny + allow specific flows` policy rather than `allow-all`. The cost of discovering a missing flow is a quick policy edit, not a production outage.
4. **Adopt the tailored-policy pattern as a team convention.** Lint-fail PRs that add a namespace without a corresponding NetworkPolicy.
5. **For vendor charts** (Longhorn, Rancher, cert-manager, MetalLB, Traefik, NeuVector, Fleet, kube-prometheus-stack) that you install during cluster bring-up, be aware you may still need `allow-all` placeholders for those specific namespaces — the [Vendor Guidance Reality Check](#vendor-guidance-reality-check) below applies in both scenarios.

If you are standing up a new cluster and installing vendor charts immediately after, jump ahead to Scenario B from the [Audit Namespaces Missing Policies](#audit-namespaces-missing-policies) step — apply it **after** vendor charts are installed but **before** the CIS profile is flipped. For pure greenfield clusters with no vendor operators yet, skip directly to [Step 4](#step-4-enable-the-cis-profile-in-rke2).

---

### Scenario B — Existing Cluster NetworkPolicy Strategy

The bulk of this section applies to existing clusters (and to new clusters after vendor charts are installed but before the profile flip).

#### Why a Blanket Default-Deny Will Break the Cluster

A `NetworkPolicy` with `policyTypes: [Ingress, Egress]` and an empty `podSelector` (no `ingress`/`egress` rules) denies **all** traffic to and from every pod in the namespace. On a Rancher + Longhorn + MetalLB + Traefik + NeuVector + Fleet cluster, applying this blindly causes:

| Namespace | What breaks under blanket default-deny |
|-----------|----------------------------------------|
| `longhorn-system` | Volume attach/detach, replica sync, CSI calls — **cluster-wide storage outage**. |
| `cert-manager` | ACME HTTP-01/DNS-01 challenges, validating webhook — **TLS issuance halts**. |
| `metallb-system` | BGP/L2 speaker peering — **LoadBalancer services stop announcing**. |
| `traefik` | Ingress → backend service traffic, ACME — **all ingress fails**. |
| `cattle-system` | Rancher server ↔ downstream agents, webhooks — **Rancher UI and cluster management dies**. |
| `cattle-fleet-*`, `cattle-capi-*`, `cattle-turtles-*`, `fleet-*` | GitOps reconciliation, cluster provisioning, bundle delivery — **Fleet/CAPI halt**. |
| `cattle-neuvector-system` | Controller ↔ enforcer ↔ scanner comms, webhook — **security policy enforcement breaks**. |
| `monitoring` | Prometheus scraping, Grafana datasources, Alertmanager clustering — **observability blind**. |
| `cattle-ui-plugin-system`, `cattle-impersonation-system`, `cattle-global-data` | Rancher UI plugins and impersonation flows — **parts of Rancher UI break**. |

#### Vendor Guidance Reality Check

It is tempting to assume the vendors of these operators ship supported NetworkPolicy manifests you can adopt. In practice, almost none do. Known state at time of writing:

| Project | Official NetworkPolicy manifests? |
|---------|-----------------------------------|
| **kube-prometheus-stack** (monitoring) | **Yes** — the upstream `kube-prometheus` project ships NetworkPolicy manifests, and the Helm chart exposes `networkPolicy.enabled` toggles per subchart. Requires validation against your scrape/remote-write/datasource topology before enabling — **not a flip-the-switch operation**. |
| Rancher (cattle-system) | No. Required ports/flows documented, no NP manifests. |
| Longhorn | No. "Networking Requirements" page lists ports only. |
| MetalLB | No. L2/BGP port docs only. |
| cert-manager | No. Webhook/egress needs mentioned in troubleshooting, no manifests. |
| Traefik | No. Helm chart has no NP templates by default. |
| NeuVector | No (ironic). Ports documented. |
| Fleet | No. |
| Cluster API / Rancher Turtles | No. |

**Conclusion:** For the CIS profile rollout, do **not** attempt to hand-author tight NetworkPolicies for vendor namespaces. That is a separate, multi-week hardening effort — see [Future Hardening](#future-hardening-technical-debt).

#### The Pragmatic Approach: allow-all Placeholders

An **`allow-all`** NetworkPolicy — one with `ingress: [{}]` and `egress: [{}]` rules — explicitly permits everything. Applied to a namespace, it:

- ✅ Satisfies CIS 5.3.2 (a policy exists).
- ✅ Preserves all existing traffic (nothing is denied).
- ❌ Provides **zero** network segmentation.
- ❌ Is not "secure" — it is a compliance placeholder.

This is the standard operational pattern on production RKE2 + Rancher clusters: apply `allow-all` everywhere first to pass the audit without outage, then incrementally replace each placeholder with a tailored policy as a separate hardening initiative.

Use an annotation on every placeholder so future operators can identify them:

```yaml
metadata:
  annotations:
    rke2.docs/note: "CIS 5.3.2 placeholder — replace with tailored policy during hardening phase"
```

> **Critical NetworkPolicy behavior:** Policies in a namespace are **additive (OR'd together)**. Adding `allow-all` to a namespace that already has a restrictive policy **neutralizes** that policy by widening the union to "everything permitted." The scripts below always skip namespaces with existing policies — do not disable those guards.

#### Audit Namespaces Missing Policies

Inventory the cluster. Exclude the four system namespaces RKE2 handles automatically:

```bash
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  case "$ns" in
    kube-system|kube-public|kube-node-lease|default) continue ;;
  esac
  count=$(kubectl get networkpolicy -n "$ns" --no-headers 2>/dev/null | wc -l)
  echo "$ns: $count NetworkPolicies"
done
```

Any namespace reporting `0 NetworkPolicies` requires a policy. On a typical Rancher-managed cluster, expect `0` for namespaces such as:

`cattle-capi-system`, `cattle-fleet-clusters-system`, `cattle-fleet-system`, `cattle-global-data`, `cattle-impersonation-system`, `cattle-local-user-passwords`, `cattle-neuvector-system`, `cattle-system`, `cattle-turtles-system`, `cattle-ui-plugin-system`, `cert-manager`, `fleet-default`, `fleet-local`, `local`, `longhorn-system`, `metallb-system`, `monitoring`, `traefik`, `cluster-fleet-local-local-<id>`, `local-p-<id>`, `p-<id>`, `u-<id>`, `user-<id>`.

The `p-*`, `u-*`, `user-*`, `local`, `local-p-*`, and `cluster-fleet-local-*` namespaces are **Rancher bookkeeping namespaces** — they hold RoleBindings, ConfigMaps, and CRs representing Rancher concepts (Projects, Users, Clusters) and typically contain **zero pods**.

#### Verify Rancher Bookkeeping Namespaces Have No Pods

A NetworkPolicy in a namespace with no pods is a runtime no-op — it has nothing to select. Confirm these namespaces are empty before treating them as safe-to-annotate with `allow-all`:

```bash
for ns in $(kubectl get ns -o name | grep -E '(^namespace/(u-|p-|user-|local$|local-p-|cluster-fleet-local-))' | cut -d/ -f2); do
  pod_count=$(kubectl get pods -n "$ns" --no-headers 2>/dev/null | wc -l)
  echo "$ns: $pod_count pods"
done
```

Expected: all report `0 pods`. If any namespace reports `>0 pods`, investigate it individually before applying the blanket rollout.

#### Canary a Single Rancher-Managed Namespace First

Rancher project and user namespaces are reconciled by Rancher controllers. Before applying `allow-all` to all of them, confirm Rancher does not strip the policy. Pick one `p-*` namespace and apply to it alone:

```bash
CANARY=$(kubectl get ns -o name | grep -m1 '^namespace/p-' | cut -d/ -f2)
echo "Canary: $CANARY"

kubectl apply -n "$CANARY" -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
  annotations:
    rke2.docs/note: "CIS 5.3.2 placeholder — replace with tailored policy during hardening phase"
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress: [{}]
  egress: [{}]
EOF
```

Wait 10–15 minutes, then verify it stuck:

```bash
kubectl get networkpolicy -n "$CANARY"
```

- **Policy still present** → Rancher is not reconciling. Proceed to the full rollout.
- **Policy gone** → Rancher is stripping it. Stop. Options at that point: apply the policy via the Rancher Project CRD's NetworkPolicy spec, or document the control as a known gap.

#### Dry-Run the Full Rollout

Before applying, confirm exactly which namespaces will be touched:

```bash
SKIP="argocd cattle-fleet-local-system kube-system kube-public kube-node-lease default"

for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  case " $SKIP " in *" $ns "*) echo "EXPLICIT-SKIP $ns"; continue ;; esac
  if kubectl get networkpolicy -n "$ns" --no-headers 2>/dev/null | grep -q .; then
    echo "AUTO-SKIP    $ns (has policies)"
  else
    echo "WILL-APPLY   $ns"
  fi
done
```

Compare `WILL-APPLY` against the earlier audit output — they must match exactly the namespaces showing `0 NetworkPolicies`. Any of your own application namespaces appearing in `WILL-APPLY` means they currently have no policies — address those intentionally rather than via `allow-all`.

Adjust the `SKIP` list to include any namespace with existing tailored policies you want to explicitly document as intentional (e.g. `argocd`, `cattle-fleet-local-system`). Explicit `SKIP` entries are belt-and-suspenders — the `kubectl get networkpolicy ... | grep -q .` guard auto-skips them regardless. Explicit listing is for documentation.

#### Apply allow-all to All Remaining Namespaces

Once the dry run looks correct, apply:

```bash
SKIP="argocd cattle-fleet-local-system kube-system kube-public kube-node-lease default"

for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  case " $SKIP " in *" $ns "*) continue ;; esac
  if kubectl get networkpolicy -n "$ns" --no-headers 2>/dev/null | grep -q .; then
    echo "skip $ns (already has policies)"
    continue
  fi
  kubectl apply -n "$ns" -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
  annotations:
    rke2.docs/note: "CIS 5.3.2 placeholder — replace with tailored policy during hardening phase"
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress: [{}]
  egress: [{}]
EOF
done
```

#### Final NetworkPolicy Verification

Every namespace except the four RKE2-managed system namespaces should now show `≥1` policy:

```bash
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  case "$ns" in kube-system|kube-public|kube-node-lease|default) continue ;; esac
  count=$(kubectl get networkpolicy -n "$ns" --no-headers 2>/dev/null | wc -l)
  echo "$ns: $count NetworkPolicies"
done
```

No namespace should report `0`. CIS 5.3.2 is now satisfied.

Also confirm no pods have been disrupted by the policy rollout:

```bash
kubectl get pods -A | grep -vE 'Running|Completed' || echo "All pods Running/Completed"
```

---

## Step 3b: Disable Default ServiceAccount Token Automount (CIS 5.1.5)

**Applies to: Both scenarios** — Scenario B runs this as a one-shot across all existing namespaces before the profile flip; Scenario A runs it whenever a new namespace is created (and should include the patch as part of any namespace-bootstrap template).

### Why This Matters

Every Kubernetes namespace ships with a `default` ServiceAccount. When a pod does not specify `spec.serviceAccountName`, it is automatically assigned the namespace's `default` SA, and its token is mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`. Any process in the pod — including a compromised sidecar or an escaped exec — can read that token and call the Kubernetes API as the `default` identity.

Even when the `default` SA has no RoleBindings today, the token is still a valid credential that can be exfiltrated, and it becomes dangerous the moment someone ever grants the `default` SA a role "just to get a workload running."

CIS control **5.1.5** ("Ensure that default service accounts are not actively used") is satisfied by setting `automountServiceAccountToken: false` on every namespace's `default` SA. RKE2's `profile: cis` handles this for the system namespaces it manages — it does **not** handle it for user namespaces or vendor-installed namespaces (Rancher, Longhorn, cert-manager, MetalLB, Traefik, monitoring, Fleet, NeuVector, or your own application namespaces).

This step closes that gap.

### Verify Current State First

Inventory which `default` ServiceAccounts still allow token automount. Anything showing `true` (or blank, which defaults to `true`) needs patching:

```bash
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  val=$(kubectl get sa default -n "$ns" -o jsonpath='{.automountServiceAccountToken}' 2>/dev/null)
  echo "$ns: automountServiceAccountToken=${val:-<unset=true>}"
done
```

Expected on an un-hardened cluster: most namespaces show `<unset=true>`. After Step 3b, all should show `false`.

### Patch All default ServiceAccounts

Apply the patch across every namespace:

```bash
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  kubectl patch serviceaccount default -n "$ns" \
    -p '{"automountServiceAccountToken": false}'
done
```

The patch is idempotent — re-running it on already-patched SAs reports `not patched` and is safe.

### Verify the Patch Took Effect

Re-run the inventory from above:

```bash
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  val=$(kubectl get sa default -n "$ns" -o jsonpath='{.automountServiceAccountToken}' 2>/dev/null)
  echo "$ns: automountServiceAccountToken=${val}"
done
```

Every namespace should now report `false`.

Also confirm no running pods are disrupted:

```bash
kubectl get pods -A | grep -vE 'Running|Completed' || echo "All pods Running/Completed"
```

Existing pods are unaffected — they already have their tokens mounted. The patch only affects **new pods** scheduled after the change that rely on the `default` SA without explicitly requesting a token.

### Important Caveat

This patch only prevents automount for pods that do **not** specify `serviceAccountName`. If a pod explicitly sets `serviceAccountName: default`, the pod spec's own `automountServiceAccountToken` field takes precedence — the pod will still get a token unless it also sets `automountServiceAccountToken: false` in its own spec.

The CIS intent is "don't *rely* on the default SA" — the patch enforces this passively. If any workload breaks after this patch, the correct fix is to create a dedicated ServiceAccount for it with only the permissions it actually needs, not to revert the patch.

For **Scenario A (new cluster)**: bake this patch into your namespace-bootstrap workflow. Every new namespace creation should be immediately followed by:

```bash
kubectl patch serviceaccount default -n <new-namespace> \
  -p '{"automountServiceAccountToken": false}'
```

Or, equivalently, ship a `ServiceAccount` manifest alongside the namespace:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: <new-namespace>
automountServiceAccountToken: false
```

---

## Step 4: Enable the CIS Profile in RKE2

### Verify config.yaml Current State First

On every node, inspect the existing (or planned) config:

```bash
cat /etc/rancher/rke2/config.yaml 2>/dev/null || echo "config.yaml does not exist yet"
```

Confirm:

- `profile:` is not already set to some conflicting value.
- `protect-kernel-defaults: false` is **not** present (it conflicts with `profile: cis`).

**Existing cluster only** — back up the file before modifying:

```bash
cp /etc/rancher/rke2/config.yaml /etc/rancher/rke2/config.yaml.bak-$(date +%Y%m%d-%H%M%S)
```

### Scenario A — New Cluster: Author config.yaml With profile: cis

On each control plane node, write the control plane config with `profile: cis` already included (this replaces the `config.yaml` from the base install guide):

```bash
mkdir -p /etc/rancher/rke2/ && sh -c 'cat > /etc/rancher/rke2/config.yaml' <<'EOF'
profile: cis
node-ip: <control-plane-private-ip>
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16

disable:
  - rke2-ingress-nginx

etcd-expose-metrics: true
secrets-encryption-provider: secretbox

kube-apiserver-arg:
  - "enable-admission-plugins=DenyServiceExternalIPs"
  - "service-account-extend-token-expiration=false"
EOF
```

> **Why `service-account-extend-token-expiration=false`:** Starting in Kubernetes 1.29, projected ServiceAccount tokens have an extended expiration behavior (up to ~1 year) intended to ease rotation. This extension weakens the CIS posture on token lifetime. For CIS compliance on RKE2 v1.29+, explicitly opt out by setting the flag to `false` so bound tokens expire at the shorter configured TTL. Required for v1.34.x.

On each worker node:

```bash
mkdir -p /etc/rancher/rke2/ && sh -c 'cat > /etc/rancher/rke2/config.yaml' <<'EOF'
profile: cis
server: https://<control-plane-private-ip>:9345
token: <control-plane-token>
node-ip: <worker-node-private-ip>

kubelet-arg:
  - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305"
  - "pod-max-pids=400"
  - "seccomp-default=true"
EOF
```

Do **not** add `protect-kernel-defaults: false` — RKE2 will refuse to start.

Then start the services for the first time as documented in [rke2-installation.md](rke2-installation.md):

```bash
# Control plane
systemctl enable --now rke2-server.service
systemctl status rke2-server.service

# Workers (after control plane is up)
systemctl enable --now rke2-agent.service
systemctl status rke2-agent.service
```

Proceed to [Step 5](#step-5-verify-cis-hardening-took-effect) to verify.

### Scenario B — Existing Cluster: Append profile: cis and Restart

On **each control plane node**, append to `/etc/rancher/rke2/config.yaml`:

```yaml
profile: cis
kube-apiserver-arg:
  - "service-account-extend-token-expiration=false"
```

> **Why `service-account-extend-token-expiration=false`:** Starting in Kubernetes 1.29, projected ServiceAccount tokens default to an extended expiration (up to ~1 year). This weakens the CIS posture on token lifetime, so RKE2's hardening guide for v1.29+ (which includes v1.34.x) requires explicitly opting out. If a `kube-apiserver-arg:` block already exists in your `config.yaml`, add the new entry under it instead of creating a second block.

On **each worker node**, append just the profile line to `/etc/rancher/rke2/config.yaml`:

```yaml
profile: cis
```

The `kube-apiserver-arg` entry is control-plane-only (workers don't run the API server). The profile flag itself is required on workers because `protect-kernel-defaults=true` and the kernel-parameter check run on the kubelet regardless of node role.

Verify the edit on each node:

```bash
grep -E 'profile|protect-kernel-defaults' /etc/rancher/rke2/config.yaml
```

Expected: `profile: cis` appears exactly once; `protect-kernel-defaults` either does not appear or is set to `true`.

Restart in order — control planes first (one at a time if you have multiple), then workers:

```bash
# On each control plane node
systemctl restart rke2-server
systemctl status rke2-server
journalctl -u rke2-server -n 50 --no-pager
```

Wait until the service reports `active (running)` and the API is reachable before moving to the next control plane node or the workers:

```bash
kubectl get nodes
```

Then restart the workers:

```bash
# On each worker node
systemctl restart rke2-agent
systemctl status rke2-agent
journalctl -u rke2-agent -n 50 --no-pager
```

If a restart fails, `journalctl` will name the exact violating kernel parameter, missing user, or other prerequisite. Fix and restart.

---

## Step 5: Verify CIS Hardening Took Effect

**Applies to: Both scenarios.**

### etcd data directory now owned by etcd:etcd

```bash
ls -ld /var/lib/rancher/rke2/server/db/etcd
```

Expected: `drwx------  etcd  etcd` (previously `root  root` on existing clusters).

### etcd process running as the etcd user

```bash
ps -eo user,pid,cmd | grep '[e]tcd' | head
```

The `USER` column must show `etcd`, not `root`.

### SecurityContext injected into the etcd static pod manifest

```bash
grep -A3 securityContext /var/lib/rancher/rke2/agent/pod-manifests/etcd.yaml
```

Expected to contain `runAsUser` and `runAsGroup` set to etcd's UID/GID.

### protect-kernel-defaults is active

```bash
ps -ef | grep kubelet | grep -o 'protect-kernel-defaults=[^ ]*' | head -1
```

Expected: `protect-kernel-defaults=true`.

### service-account-extend-token-expiration is disabled

```bash
ps -ef | grep kube-apiserver | grep -o 'service-account-extend-token-expiration=[^ ]*' | head -1
```

Expected: `service-account-extend-token-expiration=false`.

### default ServiceAccount token automount is disabled

```bash
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  val=$(kubectl get sa default -n "$ns" -o jsonpath='{.automountServiceAccountToken}' 2>/dev/null)
  [ "$val" = "false" ] || echo "NEEDS PATCH: $ns (automountServiceAccountToken=${val:-<unset=true>})"
done
```

Expected: no output (every namespace reports `false`).

### Restricted PodSecurity admission active

```bash
kubectl get --raw /api/v1/namespaces/kube-system | grep -i pod-security
```

### RKE2-managed default NetworkPolicies exist

```bash
kubectl get networkpolicy -n kube-system
kubectl get networkpolicy -n kube-public
kubectl get networkpolicy -n default
```

Each should show at least one RKE2-created policy (typically named `default-network-policy`).

### Nodes still Ready and workloads healthy

```bash
kubectl get nodes
kubectl get pods -A | grep -vE 'Running|Completed' || echo "All pods Running/Completed"
```

All nodes `Ready`; no pods stuck in non-Running states beyond normal churn.

---

## Step 6: Re-Run the RKE2 CIS Self-Assessment

The RKE2 CIS 1.11 self-assessment document is the authoritative per-control audit:

<https://docs.rke2.io/security/cis_self_assessment111>

Walk through each control in the document. Each control row lists an **Audit** command — run it and compare the result against the **Expected Result** column. Controls that were previously failing due to:

- etcd data directory ownership
- etcd process running as root
- Missing `protect-kernel-defaults`
- Missing NetworkPolicies on user namespaces

...should now pass.

Controls in the **5.x Policies** section remain operator responsibility and may still require follow-up work (PodSecurity admission enforcement on custom namespaces, RBAC review, image provenance, etc.).

---

## Future Hardening (Technical Debt)

The `allow-all` placeholders in Step 3 satisfy CIS 5.3.2 but provide no real network segmentation. They are technical debt. The hardening plan, ordered by practicality:

1. **Your own application namespaces first** — you know the traffic, the blast radius is limited, and rollback is easy.
   - Capture real traffic using Cilium Hubble, Calico flow logs, or similar.
   - Derive an allow-list from observed flows.
   - Apply in audit/dry-run mode first if your CNI supports it.
   - Canary for 24–48h, then remove the `allow-all` placeholder.

2. **`monitoring` namespace** — review `helm template ... --set <subchart>.networkPolicy.enabled=true | grep -A50 NetworkPolicy` output for the kube-prometheus-stack version in use. Validate the rendered policies cover your scrape targets, remote-write endpoints, and Grafana datasources before switching over.

3. **Infrastructure / vendor namespaces last** (`cattle-*`, `longhorn-system`, `metallb-system`, `traefik`, `cert-manager`, `cattle-neuvector-system`, `fleet-*`). These have complex, vendor-undocumented internal traffic. Hand-authoring policies here carries real risk of silent breakage on minor version bumps. Do these only with:
   - A staging cluster to validate on first.
   - An observability posture that will detect policy-induced breakage quickly.
   - Change-window discipline.

Every placeholder NetworkPolicy carries the `rke2.docs/note` annotation — use it to find remaining debt:

```bash
kubectl get networkpolicy -A -o json | \
  jq -r '.items[] | select(.metadata.annotations."rke2.docs/note") | "\(.metadata.namespace)/\(.metadata.name)"'
```

**For new clusters (Scenario A):** avoid accumulating this debt in the first place. Ship a tailored NetworkPolicy alongside every new namespace at deploy time; only fall back to `allow-all` for vendor charts that do not document their required network flows.

---

## Troubleshooting

### RKE2 refuses to start: "etcd user not found"

The `etcd` user does not exist in `/etc/passwd`, or exists only via NSS/LDAP/SSSD. Create a local user per [Step 1](#step-1-create-the-etcd-system-user-and-group). Verify with:

```bash
grep '^etcd:' /etc/passwd /etc/group
```

### RKE2 refuses to start: kernel parameter violation

The journal will name the specific sysctl. Re-apply `/etc/sysctl.d/90-kubelet.conf` and run `sysctl -p`. Verify with:

```bash
sysctl vm.panic_on_oom vm.overcommit_memory kernel.panic kernel.panic_on_oops kernel.keys.root_maxbytes
```

If a parameter resists being set, check for a conflicting entry in `/etc/sysctl.conf` or another file under `/etc/sysctl.d/` that loads later (lexical order).

### RKE2 refuses to start with profile flag: "protect-kernel-defaults conflict"

You have both `profile: cis` and `protect-kernel-defaults: false` in `config.yaml`. Inspect and fix:

```bash
grep -E 'profile|protect-kernel-defaults' /etc/rancher/rke2/config.yaml
```

Remove the `protect-kernel-defaults: false` line.

### Pods in Rancher-managed namespaces enter CrashLoopBackOff after restart

Likely the restricted PodSecurity admission profile is rejecting the pod spec. Check the namespace's enforcement labels and the pod's `securityContext`:

```bash
kubectl get ns <namespace> -o jsonpath='{.metadata.labels}' | grep pod-security
kubectl describe pod <pod> -n <namespace> | grep -A5 -i 'securityContext\|violat'
```

Rancher system pods generally handle this correctly; custom workloads may need adjustment.

### NetworkPolicy canary was stripped by Rancher

Rancher is actively reconciling that namespace. Re-check:

```bash
kubectl get networkpolicy -n <canary-namespace> --watch
```

If policies disappear within minutes of being applied, apply the policy via the Rancher Project NetworkPolicy mechanism instead, or document the affected namespaces as a known CIS 5.3.2 gap with compensating controls (CNI-level isolation, namespace-level RBAC, etc.).

### Nodes stuck NotReady after agent restart

Check on the affected node:

```bash
journalctl -u rke2-agent -n 200 --no-pager
```

The most common causes after a CIS profile flip are: missing sysctls on that node specifically, or CNI pods failing PodSecurity admission. Correct locally and restart the agent.
