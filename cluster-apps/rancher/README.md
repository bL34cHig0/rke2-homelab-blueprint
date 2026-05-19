# Rancher Production Deployment on RKE2

## Table of Contents

- [Architecture Overview](#architecture-overview)
  - [TLS Termination Model](#tls-termination-model)
  - [Manifest Files](#manifest-files)
- [Prerequisites](#prerequisites)
- [Deployment Steps](#deployment-steps)
  - [Step 1: Create the Namespace](#step-1-create-the-namespace)
  - [Step 2: Install cert-manager](#step-2-install-cert-manager)
  - [Step 3: Add Rancher Helm Repositories](#step-3-add-rancher-helm-repositories)
  - [Step 4: Choose a Rancher Version](#step-4-choose-a-rancher-version)
  - [Step 5: Install Rancher with Helm](#step-5-install-rancher-with-helm)
    - [What `tls=external` actually does](#what-tlsexternal-actually-does)
  - [Step 6: Apply TLS Certificate](#step-6-apply-tls-certificate)
  - [Step 7: Apply Traefik Middlewares and IngressRoute](#step-7-apply-traefik-middlewares-and-ingressroute)
  - [Step 8: Configure DNS](#step-8-configure-dns)
  - [Step 9: Initial Login and Bootstrap Password](#step-9-initial-login-and-bootstrap-password)
- [Authentication](#authentication)
  - [Choosing Between OAuth App and GitHub App](#choosing-between-oauth-app-and-github-app)
  - [Option A: GitHub OAuth App](#option-a-github-oauth-app)
    - [A.1 Create the OAuth App in GitHub](#a1-create-the-oauth-app-in-github)
    - [A.2 Configure in Rancher](#a2-configure-in-rancher)
  - [Option B: GitHub App](#option-b-github-app)
    - [B.1 Create the GitHub App](#b1-create-the-github-app)
    - [B.2 Install the App on the Organization](#b2-install-the-app-on-the-organization)
    - [B.3 Configure in Rancher](#b3-configure-in-rancher)
  - [Site Access (User Authorization Scope)](#site-access-user-authorization-scope)
  - [Vetting Authorized Users](#vetting-authorized-users)
- [Switching Helm Chart Repositories](#switching-helm-chart-repositories)
  - [Switch from `latest` to `stable` (or vice versa)](#switch-from-latest-to-stable-or-vice-versa)
- [Upgrades](#upgrades)
- [Maintenance and Troubleshooting](#maintenance-and-troubleshooting)
  - [Verify Rancher Pods](#verify-rancher-pods)
  - [Inspect the Helm Release](#inspect-the-helm-release)
  - [Reset the Bootstrap Password](#reset-the-bootstrap-password)
  - [Certificate Issues](#certificate-issues)
  - [Ingress Not Routing](#ingress-not-routing)
  - [Uninstall Rancher](#uninstall-rancher)

---

## Architecture Overview

Rancher is deployed in the `cattle-system` namespace via the official Helm chart. External traffic terminates TLS at **Traefik** using a wildcard certificate issued by **cert-manager** (Let's Encrypt), then is forwarded over plain HTTP to the Rancher Service inside the cluster.

| Component | Kind | Replicas | Purpose |
|---|---|---|---|
| **rancher** | Deployment | 2 | Rancher control-plane API and UI |
| **cert-manager** | Helm release | — | Issues and renews TLS certificates from Let's Encrypt |
| **Traefik IngressRoute** | CRD | — | Terminates TLS, applies headers and rate-limit middlewares, routes to the Rancher Service |

### TLS Termination Model

This deployment uses `--set tls=external`. Rancher itself does **not** terminate TLS or run an Ingress controller — Traefik does both. The Rancher Service is exposed on port `80` (HTTP) inside the cluster; Traefik handles HTTPS and forwards `X-Forwarded-Proto: https` via the `default-headers` middleware.

This is the recommended pattern when you already run an ingress controller in front of Rancher. It keeps certificate management consistent with the rest of the cluster (cert-manager + a single wildcard cert) and avoids running two ingress controllers.

### Manifest Files

| File | Description |
|---|---|
| `rancher-certificate.yaml` | cert-manager `Certificate` for the wildcard `*.<your-domain>` (writes the `default-cert-production-tls` secret consumed by Traefik) |
| `default-headers.yaml` | Traefik `Middleware` resources — security headers (HSTS, XSS filter, frame options, `X-Forwarded-Proto`) and rate limiting |
| `ingress.yaml` | Traefik `IngressRoute` — routes `rancher.<your-domain>` and `www.rancher.<your-domain>` to the Rancher Service on port 80 |

---

## Prerequisites

Login with the service account on the control plane via `ssh`. Switch to the root user — `sudo su` then `cd ~`.

Download the repo on the control plane and navigate to the `rancher` directory. Use `git clone` or the `Download Zip` button to download the repo.

Replace **placeholders** for hostnames and credentials with valid ones. The hostname configured at install time **must match** the host in `ingress.yaml` and the SAN(s) covered by `rancher-certificate.yaml`.

**Caveat**: All Rancher manifests must be applied in the `cattle-system` namespace. Before generating credentials with CLI tools, ensure credentials are not being logged in shell history by adding the following to `~/.bashrc` (if not already present):

```bash
# use nano to open the root user's bashrc file
nano .bashrc

# add the following configuration below the history length settings
# display history commands timestamp
export HISTTIMEFORMAT="%F %T "

# ignore sensitive commands
export HISTIGNORE="echo*:*password*:*secret*:*token*:*htpasswd*:*base64*:openssl*"

# source bashrc file to load the configuration
source .bashrc
```

Generate a strong, unique bootstrap password with a password manager. Avoid **special characters** and **punctuation** to prevent them from breaking the Helm install. Use alphanumeric passwords (>= 18 characters) and store them securely.

You will also need:

- An RKE2 cluster with at least **2 worker nodes** (Rancher runs 2 replicas with anti-affinity).
- A working **Traefik** ingress (see [traefik](../traefik/) folder).
- Helm 3.x installed on the control plane.

---

## Deployment Steps

### Step 1: Create the Namespace

```bash
# switch to root user
sudo su
cd ~

# clone the repo and change to the rancher directory
git clone <repo-url>
cd rancher

# create the cattle-system namespace
kubectl create namespace cattle-system

# verify all manifest files are intact
ls
```

### Step 2: Install cert-manager

Rancher relies on cert-manager to issue TLS certificates. Even with `tls=external`, cert-manager is required in this cluster because the Traefik-facing certificate (`default-cert-production-tls`) is itself managed by cert-manager.

```bash
# add the jetstack helm repo
helm repo add jetstack https://charts.jetstack.io
helm repo update

# install cert-manager with CRDs (skip this step if cert-manager is already installed cluster-wide)
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true

# verify all cert-manager pods are running
kubectl get pods -n cert-manager
```

> Note: If cert-manager is already installed (e.g. for other apps in this repo), skip this step. Confirm with `helm list -A | grep cert-manager`.

### Step 3: Add Rancher Helm Repositories

Rancher publishes three chart repositories:

| Repo | URL | Use case |
|---|---|---|
| `rancher-stable` | `https://releases.rancher.com/server-charts/stable` | **Recommended for production.** Versions are promoted here only after at least one month of `latest` release exposure with no critical bugs. |
| `rancher-latest` | `https://releases.rancher.com/server-charts/latest` | Newest minor and patch releases. Suitable for testing and non-production. |
| `rancher-alpha` | `https://releases.rancher.com/server-charts/alpha` | Pre-releases. Not for production. |

Add both production candidates so you can switch later without re-adding repos:

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
```

### Step 4: Choose a Rancher Version

Pin the chart version explicitly — never install Rancher without `--version`. This makes upgrades and rollbacks deterministic.

List the available versions in each repo:

```bash
# list all chart versions in the stable repo (newest first)
helm search repo rancher-stable/rancher --versions

# list all chart versions in the latest repo
helm search repo rancher-latest/rancher --versions

# show only the most recent stable version
helm search repo rancher-stable/rancher
```

Cross-reference the chart version with the [Rancher releases page](https://github.com/rancher/rancher/releases) to confirm the corresponding Rancher app version, the supported Kubernetes versions, and any breaking-change notes.

For production, select the **highest version available in `rancher-stable`** that is compatible with your RKE2 Kubernetes version.

### Step 5: Install Rancher with Helm

Install from `rancher-stable` (recommended for production). Replace placeholders:

- `<hostname-in-ingress-yaml>` — must match the `Host(...)` rule in `ingress.yaml` (e.g. `rancher.<your-domain>`).
- `<strong-password>` — initial admin password, alphanumeric, 18+ characters.
- `<stable-version>` — the chart version chosen in Step 4 (e.g. `2.9.5`).

```bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=<hostname-in-ingress-yaml> \
  --set replicas=2 \
  --set bootstrapPassword=<strong-password> \
  --set tls=external \
  --set ingress.enabled=false \
  --set service.type=ClusterIP \
  --version=<stable-version>
```

Key flags explained:

| Flag | Purpose |
|---|---|
| `--set hostname=...` | The external hostname users browse to. Rancher uses this for redirects and link generation. Must match the IngressRoute host. |
| `--set replicas=2` | Runs two Rancher pods for HA. Requires at least two schedulable nodes. |
| `--set bootstrapPassword=...` | Initial password for the local `admin` user. You will be prompted to change it on first login. |
| `--set tls=external` | Tells Rancher that TLS is terminated by something in front of it (Traefik, in this case). See the explanation below. |
| `--set ingress.enabled=false` | Prevents the Rancher chart from creating its own Kubernetes `Ingress` resource. Traffic reaches Rancher via the Traefik `IngressRoute` in `ingress.yaml` instead, so a chart-managed Ingress would be redundant (and could conflict if a different ingress controller picked it up). |
| `--set service.type=ClusterIP` | Exposes the Rancher Service only inside the cluster. The chart sometimes defaults this differently depending on `tls`/`ingress` combinations; setting it explicitly guarantees no `NodePort` or `LoadBalancer` is provisioned, since external access goes exclusively through Traefik. |
| `--version=...` | Pins the chart version. Always set explicitly. |

#### What `tls=external` actually does

The Rancher chart has three TLS modes, controlled by the `tls` value:

| Mode | Behavior |
|---|---|
| `tls=ingress` (default) | The chart creates a Kubernetes `Ingress` resource and terminates TLS at the Ingress. The certificate source is controlled by `ingress.tls.source` (`rancher` self-signed, `letsEncrypt`, or `secret`). Requires cert-manager when source is `rancher` or `letsEncrypt`. |
| `tls=external` | Rancher itself does **not** terminate TLS. The chart skips the Ingress and TLS-related machinery entirely. The Rancher Service is exposed on **HTTP port 80** inside the cluster, and an external component (load balancer, ingress controller, reverse proxy) is expected to terminate HTTPS and forward plain HTTP to that Service. Rancher trusts the `X-Forwarded-Proto: https` header to know the original request was HTTPS — this header is set by the `default-headers` Traefik middleware in `default-headers.yaml`. |
| `tls=` (unset / empty in newer chart versions) | Some chart versions allow this for HTTP-only setups without proxy headers. Not used here. |

In this deployment, `tls=external` is correct because:

1. Traefik already runs in front of every other workload in this cluster and handles TLS termination for them.
2. The wildcard certificate `default-cert-production-tls` is issued by cert-manager and consumed by Traefik (see `rancher-certificate.yaml` and `ingress.yaml`) — the same certificate management story as the rest of the cluster.
3. Rancher's own ingress would either duplicate Traefik's role or conflict with it, depending on what `IngressClass` it picked.

Without `tls=external`, Rancher would try to obtain its own cert (often via cert-manager) and create its own Ingress, leading to two layers of TLS termination and two competing certificates for the same hostname.

Watch the rollout until both replicas are ready:

```bash
kubectl -n cattle-system rollout status deploy/rancher
kubectl get pods -n cattle-system
```

### Step 6: Apply TLS Certificate

```bash
kubectl apply -f rancher-certificate.yaml

# verify the certificate is issued (READY=True)
kubectl get certificate -n cattle-system
kubectl describe certificate -n cattle-system default-cert-production
```

If the certificate stays in a `False` ready state for more than a minute or two, see [Certificate Issues](#certificate-issues) below.

### Step 7: Apply Traefik Middlewares and IngressRoute

```bash
kubectl apply -f default-headers.yaml
kubectl apply -f ingress.yaml

# verify resources
kubectl get middleware -n cattle-system
kubectl get ingressroute -n cattle-system
```

### Step 8: Configure DNS

Navigate to Cloudflare or your DNS provider.

Point an `A` record at the designated ingress node's public IP with the name matching the `hostname` set during Helm install and the `Host(...)` in `ingress.yaml`. If you already have a wildcard record (e.g. `*.example.com`) covering that hostname, no additional record is needed — the wildcard suffices.

**Note**: wildcards match exactly one label and do **not** cover the apex domain or deeper subdomains. For example, `*.example.com` covers `rancher.example.com` but neither `example.com` nor `rancher.apps.example.com` — those need their own records.

### Step 9: Initial Login and Bootstrap Password

Once DNS resolves, browse to `https://<hostname>` and log in with username `admin` and the bootstrap password set in Step 5. You will be required to set a new password on first login. Store the new password in a password manager.

If you forget the bootstrap password before first login, see [Reset the Bootstrap Password](#reset-the-bootstrap-password).

---

## Authentication

Out of the box, Rancher uses its built-in **local** auth provider — only the `admin` user (and any local users you create) can log in. For team use, replace this with a centralized identity provider so access can be granted and revoked at the source rather than per-Rancher-instance.

This guide covers GitHub-based authentication, which Rancher supports in two flavors:

- **GitHub OAuth App** — the original, simpler integration. Uses a personal/org-owned OAuth App.
- **GitHub App** — a newer alternative with finer-grained permissions, optional installation per organization, and the ability to use a private key for stronger app-level auth.

References:

- [Configure GitHub (OAuth App)](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/authentication-permissions-and-global-configuration/authentication-config/configure-github)
- [Configure GitHub App](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/authentication-permissions-and-global-configuration/authentication-config/configure-github-app)

### Choosing Between OAuth App and GitHub App

| Criterion | OAuth App | GitHub App |
|---|---|---|
| Setup complexity | Simpler | A few more fields (App ID, private key) |
| Permission granularity | User-scoped OAuth scopes | Fine-grained app permissions |
| Organization installation | Implicit (org members granted via OAuth scope) | Explicitly installed on each org |
| Recommended for | Small teams, single org | Multi-org setups, stricter permission audits |

If your team is on a single GitHub organization and you want the fastest path, **OAuth App** is fine. If you manage access across multiple orgs or want explicit per-org install control, prefer **GitHub App**.

### Option A: GitHub OAuth App

#### A.1 Create the OAuth App in GitHub

1. In GitHub, navigate to one of:
   - **Personal account**: Settings → Developer settings → OAuth Apps → **New OAuth App**
   - **Organization** (recommended for shared use): Org settings → Developer settings → OAuth Apps → **New OAuth App**
2. Fill in:
   - **Application name**: e.g. `Rancher`
   - **Homepage URL**: `https://<rancher-hostname>` (the hostname you set in Step 5)
   - **Authorization callback URL**: `https://<rancher-hostname>/verify-auth`
3. Click **Register application**.
4. On the resulting page, copy the **Client ID** and click **Generate a new client secret** — copy the secret immediately, it is shown only once.

#### A.2 Configure in Rancher

1. Log in to Rancher as `admin` (local).
2. Go to ☰ → **Users & Authentication** → **Auth Provider**.
3. Select **GitHub**.
4. Paste the **Client ID** and **Client Secret** from the previous step.
5. Click **Authenticate with GitHub**. A GitHub OAuth flow opens — approve the requested scopes with a GitHub user that is a member of the organization(s) you want to grant access to. This same user becomes the first GitHub-authenticated Rancher user and is granted admin.
6. Once redirected back, GitHub auth is enabled. Continue to [Site Access](#site-access-user-authorization-scope) below to lock down who is allowed in.

> Note: The user that completes the OAuth flow in step 5 must have **org owner** or sufficient permission to approve the OAuth App for the org if the org enforces OAuth App access restrictions. Otherwise org members will not be visible to Rancher.

### Option B: GitHub App

#### B.1 Create the GitHub App

1. In GitHub, navigate to **Org settings → Developer settings → GitHub Apps → New GitHub App** (or the personal-account equivalent).
2. Fill in:
   - **GitHub App name**: e.g. `Rancher-<org-name>`
   - **Homepage URL**: `https://<rancher-hostname>`
   - **Callback URL**: `https://<rancher-hostname>/verify-auth`
   - **Webhook → Active**: **uncheck** (Rancher does not consume webhooks).
3. Set **Repository permissions** to **No access** (Rancher only needs identity, not repo data) unless you have a specific reason to grant more.
4. Set **Account / Organization permissions**:
   - **Members**: Read-only (required to enumerate org members for "members of org" site-access mode).
   - **Email addresses**: Read-only.
5. **Where can this GitHub App be installed?**: choose **Only on this account** for a single org, or **Any account** if you intend to install it on multiple orgs.
6. Click **Create GitHub App**.
7. On the App's settings page:
   - Copy the **App ID**.
   - Click **Generate a new client secret** and copy it.
   - Click **Generate a private key** — a `.pem` file downloads. Keep this file safe; you will paste its contents into Rancher.

#### B.2 Install the App on the Organization

1. In the App's settings, click **Install App** in the left sidebar.
2. Click **Install** next to the target organization.
3. Choose **All repositories** or specific repos — for auth-only use, the choice doesn't matter (no repo permissions are requested).
4. Confirm the installation.

#### B.3 Configure in Rancher

1. Log in to Rancher as `admin` (local).
2. Go to ☰ → **Users & Authentication** → **Auth Provider** → **GitHub**.
3. Toggle the mode to **GitHub App** (the page exposes additional fields when this is selected).
4. Paste:
   - **Client ID**
   - **Client Secret**
   - **App ID**
   - **Private Key** — open the downloaded `.pem` file and paste its full contents, including the `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----` lines.
5. Click **Authenticate with GitHub** and complete the OAuth flow as in Option A. The authenticating user becomes the first GitHub-authenticated Rancher user and is granted admin.

### Site Access (User Authorization Scope)

After enabling GitHub auth, scroll down on the same page to **Site Access** (also labelled **Configure who has access to Rancher**). The three options are:

| Setting | Effect |
|---|---|
| **Allow any valid Users** | Any GitHub user who can log in via OAuth gets a Rancher account. Most permissive — avoid in production. |
| **Allow members of Clusters, Projects, plus Authorized Users and Organizations** | Existing Rancher users (already in any cluster/project) plus the explicit allow-list below can log in. New unlisted GitHub users cannot. |
| **Restrict access to only Authorized Users and Organizations** | Only the explicit allow-list below may log in. The strictest option. |

**Use the middle option — "Allow members of Clusters, Projects, plus Authorized Users and Organizations".** This avoids a single point of failure: it includes both the explicit allow-list **and** anyone already mapped into a cluster or project, so a single misconfiguration of the allow-list does not lock out an entire team that already has legitimate workload-level access.

In the **Authorized Users and Organizations** field below the radio selector:

1. Type a GitHub **organization name** to grant login to all members of that org.
2. Type a GitHub **username** to grant login to that specific user.
3. Click **Save**.

### Vetting Authorized Users

Authorization scope decides **who can log in**, but everyone who logs in gets at least the **Standard User** global role by default — they can create their own clusters, view their own projects, and consume cluster resources where they have been added. Authenticated users are **not powerless**, so vetting matters even for the middle access option.

Before adding a username or organization to the **Authorized Users and Organizations** list:

- **Confirm the GitHub identity belongs to the person you think it does.** Out-of-band verification (e.g. ask them in chat or in person) — usernames can be transferred or impersonated by display-name spoofing.
- **Confirm the org has 2FA enforced.** If the org allows members without 2FA, attackers who phish a member's GitHub creds can pivot into Rancher.
- **Grant the minimum global role.** New users default to **Standard User**. Downgrade to **User-Base** (read-only Rancher API + their own settings) for users who only need cluster-scoped access — set per-user via Users & Authentication → Users → [user] → Global Permissions.
- **Assign cluster/project roles explicitly.** Standard User does **not** automatically get access to any cluster or project. Add users to specific clusters/projects via the cluster's Members tab with the least privilege required (`Cluster Owner`, `Cluster Member`, `Project Owner`, `Project Member`, `Read Only`, etc.).
- **Audit periodically.** Review Users & Authentication → Users for accounts that haven't logged in recently or no longer need access. Disable rather than delete to preserve audit trails.

> Note: If you ever need to revoke access for a user who was authorized via an organization membership rather than by username, removing them from the GitHub org does **not** immediately log them out of Rancher — Rancher caches group membership for the duration of their session token. Disable the user in Rancher for immediate revocation.

---

## Switching Helm Chart Repositories

Rancher's chart repos (`rancher-stable`, `rancher-latest`, `rancher-alpha`) are independent. Switching between them requires a Helm upgrade against a different repo — Helm will not pick up a new repo automatically.

This procedure follows Rancher's official guide: [Switching to a Different Helm Chart Repository](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/choose-a-rancher-version#switching-to-a-different-helm-chart-repository).

### Switch from `latest` to `stable` (or vice versa)

1. **Confirm the currently installed release and chart version**:

   ```bash
   helm list -n cattle-system
   helm get values rancher -n cattle-system
   ```

   Note the `CHART` column (e.g. `rancher-2.10.1`) — this is the version currently deployed.

2. **Make sure both repos are present and refreshed**:

   ```bash
   helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
   helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
   helm repo update
   ```

3. **Find the equivalent (or newer) version in the target repo**:

   ```bash
   helm search repo rancher-stable/rancher --versions | head -n 10
   ```

   Pick a version that is **equal to or newer** than what is currently installed. Downgrading is not supported.

4. **Get the current values so they are preserved on upgrade**:

   ```bash
   helm get values rancher -n cattle-system -o yaml > rancher-current-values.yaml
   ```

5. **Upgrade against the new repo, pinning the new version**:

   ```bash
   helm upgrade rancher rancher-stable/rancher \
     --namespace cattle-system \
     --version=<target-version> \
     -f rancher-current-values.yaml
   ```

6. **Watch the rollout** and verify the new chart version appears in `helm list`:

   ```bash
   kubectl -n cattle-system rollout status deploy/rancher
   helm list -n cattle-system
   ```

7. **Securely remove the saved values file** once the upgrade is verified — it may contain the bootstrap password if it was set in the original install:

   ```bash
   shred -u rancher-current-values.yaml
   ```

> Note: Switching from `rancher-stable` to `rancher-alpha` is **not recommended**. The Rancher docs warn that alpha releases may contain features that are removed before becoming stable, and downgrading away from alpha is not supported.

---

## Upgrades

Routine in-repo upgrades (e.g. `rancher-stable` 2.9.5 → 2.9.6) follow the same pattern as a repo switch but stay on the same repo:

```bash
# refresh repo metadata
helm repo update

# check for newer versions
helm search repo rancher-stable/rancher

# preserve current values
helm get values rancher -n cattle-system -o yaml > rancher-current-values.yaml

# upgrade
helm upgrade rancher rancher-stable/rancher \
  --namespace cattle-system \
  --version=<new-version> \
  -f rancher-current-values.yaml

# verify
kubectl -n cattle-system rollout status deploy/rancher
helm list -n cattle-system

# remove the temporary values file
shred -u rancher-current-values.yaml
```

Always read the [release notes](https://github.com/rancher/rancher/releases) for the target version before upgrading. Pay particular attention to required Kubernetes version ranges and any noted manual migration steps.

---

## Maintenance and Troubleshooting

### Verify Rancher Pods

```bash
kubectl get pods -n cattle-system -l app=rancher
kubectl logs -n cattle-system deployment/rancher
kubectl describe pod -n cattle-system -l app=rancher
```

### Inspect the Helm Release

```bash
# show installed chart version, namespace, and status
helm list -n cattle-system

# show the values currently applied to the release
helm get values rancher -n cattle-system

# show the full rendered manifests
helm get manifest rancher -n cattle-system
```

### Reset the Bootstrap Password

If you lose the bootstrap password before first login, reset it from inside a Rancher pod:

```bash
kubectl exec -n cattle-system -t deployment/rancher -- reset-password
```

The command prints a new randomly generated password. Use it to log in, then change it through the UI.

### Certificate Issues

If `kubectl get certificate -n cattle-system` shows `READY=False`:

```bash
# inspect the certificate events for the specific failure reason
kubectl describe certificate -n cattle-system default-cert-production

# check the cert-manager controller logs
kubectl logs -n cert-manager deployment/cert-manager --tail=200

# verify the ClusterIssuer is healthy
kubectl get clusterissuer letsencrypt-production
kubectl describe clusterissuer letsencrypt-production
```

Common causes:

- DNS for the requested SAN is not resolving to the cluster ingress IP.
- Let's Encrypt rate limits hit (5 duplicate certs per week per registered domain).
- ClusterIssuer is misconfigured or its solver lacks credentials (e.g. Cloudflare API token).

### Ingress Not Routing

```bash
# confirm the IngressRoute and middlewares exist in the right namespace
kubectl get ingressroute,middleware -n cattle-system

# confirm the rancher Service is up and selecting pods
kubectl get svc rancher -n cattle-system
kubectl get endpoints rancher -n cattle-system

# check Traefik logs for routing errors
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik --tail=200
```

If the endpoints list is empty, the Service selector does not match any Rancher pods — verify with `kubectl get pods -n cattle-system --show-labels`.

### Uninstall Rancher

> **Warning**: This deletes all Rancher data including managed cluster registrations. Back up first if needed.

```bash
helm uninstall rancher -n cattle-system

# remove leftover CRDs and namespaces
kubectl delete namespace cattle-system
kubectl get crd | grep cattle.io | awk '{print $1}' | xargs kubectl delete crd
```

For the official, fully thorough cleanup procedure (especially when migrating to a fresh install), see Rancher's [Rancher is No Longer Needed](https://ranchermanager.docs.rancher.com/faq/rancher-is-no-longer-needed) FAQ, which points to the [rancher-cleanup](https://github.com/rancher/rancher-cleanup) tool. (The older System Tools approach was deprecated in June 2022.)
