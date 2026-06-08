# Kubernetes Workload Hardening Blueprint

This guide collects the security defaults that produce a hardened, production-grade workload on a Kubernetes cluster. Treat it as a baseline — adopt the controls that fit your threat model and intentionally relax the rest, understanding the tradeoff. Each section explains *why* the control exists so you can make that call.

## Table of Contents

- [1. Pod Identity & Service Accounts](#1-pod-identity--service-accounts)
  - [1.1 Custom Service Accounts](#11-custom-service-accounts)
  - [1.2 API Token Mounting](#12-api-token-mounting)
- [2. Pod Security Context](#2-pod-security-context)
  - [2.1 Non-Root Enforcement](#21-non-root-enforcement)
  - [2.2 Seccomp Profile](#22-seccomp-profile)
- [3. Container Security Context](#3-container-security-context)
  - [3.1 Privilege Controls](#31-privilege-controls)
  - [3.2 Linux Capabilities](#32-linux-capabilities)
  - [3.3 Read-Only Root Filesystem](#33-read-only-root-filesystem)
- [4. Resource Limits](#4-resource-limits)
- [5. Health Probes](#5-health-probes)
  - [5.1 Probe Types](#51-probe-types)
  - [5.2 Probe Method Guidelines](#52-probe-method-guidelines)
  - [5.3 Guidelines](#53-guidelines)
- [6. Image Security](#6-image-security)
  - [6.1 Image Sources](#61-image-sources)
  - [6.2 Image Build](#62-image-build)
- [7. Network Segmentation & Ingress](#7-network-segmentation--ingress)
  - [7.1 Default Deny Posture](#71-default-deny-posture)
  - [7.2 Microsegmentation](#72-microsegmentation)
  - [7.3 Ingress Rules](#73-ingress-rules)
  - [7.4 Egress Rules](#74-egress-rules)
  - [7.5 Label Selectors](#75-label-selectors)
  - [7.6 Ingress Controller & TLS](#76-ingress-controller--tls)
  - [7.7 Rate Limiting & IP Spoofing Prevention](#77-rate-limiting--ip-spoofing-prevention)
  - [7.8 Cloudflare Proxy: Extracting Real Client IPs](#78-cloudflare-proxy-extracting-real-client-ips)
    - [7.8.1 Automated IP Range Sync](#781-automated-ip-range-sync)
  - [7.9 Verification](#79-verification)
- [8. Pre-deployment Hardening Checklist](#8-pre-deployment-hardening-checklist)

---

## 1. Pod Identity & Service Accounts

### 1.1 Custom Service Accounts

Give every workload its own dedicated ServiceAccount and don't reuse the `default` ServiceAccount in any namespace.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <app-name>
automountServiceAccountToken: false
```

**Why:** The default SA is shared across every pod in the namespace. A compromised pod using the default SA inherits any RBAC permissions ever granted to it — which may have been widened for an unrelated workload.

### 1.2 API Token Mounting

Set `automountServiceAccountToken: false` on both the ServiceAccount and the Pod spec unless the workload genuinely needs Kubernetes API access.

```yaml
# ServiceAccount level
automountServiceAccountToken: false

# Pod spec level (takes precedence)
spec:
  automountServiceAccountToken: false
```

If a workload does need API access (an operator, a sidecar that watches ConfigMaps, etc.), bind a dedicated Role or ClusterRole with the minimum verbs and resources it needs. Never grant `*` (wildcard) verbs or resources.

---

## 2. Pod Security Context

Define a `securityContext` at the pod level on every pod:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: <uid>        # Should match the USER in the Dockerfile
    runAsGroup: <gid>       # Should match the group in the Dockerfile
    fsGroup: <gid>          # Needed if the pod writes to mounted volumes
    seccompProfile:
      type: RuntimeDefault
```

### 2.1 Non-Root Enforcement

| Field | Recommended value | Notes |
|-------|---------------|-------|
| `runAsNonRoot` | `true` | Kubernetes rejects the pod if the container attempts to run as UID 0 |
| `runAsUser` | Non-zero UID | Should align with the unprivileged user created in the Dockerfile |
| `runAsGroup` | Non-zero GID | Should align with the group in the Dockerfile |

**Dockerfile pairing:** Create a non-root user in the Dockerfile and switch to it before `CMD`/`ENTRYPOINT`:

```dockerfile
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 appuser
USER appuser
```

### 2.2 Seccomp Profile

Use `seccompProfile.type: RuntimeDefault` at minimum. This applies the container runtime's default seccomp filter, blocking dangerous syscalls (`reboot`, `mount`, `kexec_load`, `ptrace`, etc.) while allowing normal application operations.

Custom seccomp profiles can be used for further restriction but are rarely necessary for typical workloads.

---

## 3. Container Security Context

Define a `securityContext` at the container level on every container:

```yaml
containers:
  - name: <container>
    securityContext:
      allowPrivilegeEscalation: false
      privileged: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
```

### 3.1 Privilege Controls

| Field | Recommended value | Purpose |
|-------|---------------|---------|
| `privileged` | `false` | Blocks full host access. A privileged container bypasses all namespace isolation and can access host devices. |
| `allowPrivilegeEscalation` | `false` | Blocks setuid/setgid binaries. Even if an attacker uploads a setuid binary, it can't escalate privileges. |

### 3.2 Linux Capabilities

Drop all capabilities by default:

```yaml
capabilities:
  drop:
    - ALL
```

If the workload genuinely needs a specific capability, add it back explicitly and comment why:

```yaml
capabilities:
  drop:
    - ALL
  add:
    - NET_BIND_SERVICE   # Required: app binds to port 80
```

**Common capabilities and when to add them:**

| Capability | When needed |
|------------|-------------|
| `NET_BIND_SERVICE` | Binding to ports below 1024 |
| `CHOWN` | Changing file ownership on shared volumes |
| `NET_RAW` | Raw socket access (network diagnostics tools only) |
| `SYS_PTRACE` | Debugging tools, profilers (never in production) |

Application workloads almost never need any capability beyond these.

### 3.3 Read-Only Root Filesystem

Set `readOnlyRootFilesystem: true` and scope writable paths through `emptyDir` volumes:

```yaml
volumeMounts:
  - name: tmp
    mountPath: /tmp
  - name: app-cache
    mountPath: /app/cache
volumes:
  - name: tmp
    emptyDir: {}
  - name: app-cache
    emptyDir: {}
```

**Guidelines for writable volumes:**

- Mount only the specific paths the application writes to — never mount `/` or `/app` as writable
- Use `emptyDir` for caches and temp files (ephemeral, lost on pod restart)
- Use `emptyDir` with `sizeLimit` for volumes that could grow unbounded:
  ```yaml
  emptyDir:
    sizeLimit: 100Mi
  ```
- Avoid `hostPath` volumes — they expose host directories to the container and break portability between nodes

---

## 4. Resource Limits

Define resource requests and limits on every container — they let the scheduler place pods sensibly and stop a runaway process from starving the node.

The recommended baseline:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    memory: 256Mi    # Same as request → Guaranteed QoS
    # CPU limit intentionally omitted — see below
```

Two patterns underpin this shape:

**Memory request and limit should usually be equal.** Equal values place the pod in Kubernetes' `Guaranteed` QoS class — it's the last to be evicted under node memory pressure, and the scheduler reserves the full amount up front. A pod with `request: 100Mi` and `limit: 500Mi` can land on a node with only ~150Mi of spare memory and get OOM-killed the moment it grows past what's actually available. Equal request and limit eliminates that surprise.

**Consider skipping the CPU limit on latency-sensitive workloads.** CPU limits throttle a container even when the node has idle CPU to spare, which hurts tail latency under bursty load. The pattern is to keep the CPU *request* (so the scheduler still reserves capacity and bin-packs sensibly) but omit the CPU *limit*, letting the container burst into spare CPU when it's there. Memory is different — keep memory limits in place. Without a memory limit a leaky container can OOM the whole node and take other pods down with it; with a limit, only the offending container is killed.

Skipping the CPU limit works best when:

- The workload is latency-sensitive (web servers, API gateways, real-time services) and benefits from burst capacity
- Requests are sized accurately, so the scheduler still packs the node correctly and per-pod fair-share is meaningful
- You trust the workload — a buggy or hostile container without a CPU limit can monopolize a node's CPU

For untrusted workloads, batch jobs, or anything where predictable resource usage matters more than peak performance, keep both limits in place.

**Further reading:** Robusta's [Stop Using CPU Limits on Kubernetes](https://home.robusta.dev/blog/stop-using-cpu-limits) makes the case for this pattern and notes that Tim Hockin — one of the original Kubernetes maintainers at Google — has recommended the same for years.

---

## 5. Health Probes

Define startup, liveness, and readiness probes on every container. Without them, Kubernetes can't tell unhealthy pods apart from healthy ones — failed pods aren't restarted, and Services route traffic to pods that aren't ready to serve.

```yaml
containers:
  - name: <container>
    startupProbe:
      # Probe definition
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      # Probe definition
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    readinessProbe:
      # Probe definition
      periodSeconds: 5
      timeoutSeconds: 5
      failureThreshold: 3
```

### 5.1 Probe Types

| Probe | Purpose | Consequence of omission |
|-------|---------|------------------------|
| **Startup** | Gates liveness/readiness until the container is initialized | Slow-starting containers are killed prematurely by the liveness probe |
| **Liveness** | Detects deadlocked or hung processes | Failed pods remain running indefinitely, serving errors or hanging |
| **Readiness** | Controls whether the pod receives traffic via Services | Unhealthy pods continue receiving traffic, causing user-facing errors |

### 5.2 Probe Method Guidelines

| Workload type | Recommended method | Example |
|---------------|-------------------|---------|
| HTTP services | `httpGet` on a known endpoint | `httpGet: { path: /healthz, port: 8080 }` |
| Databases | `exec` using the database's built-in health check tool | `exec: { command: [mariadb-admin, ping, -h, localhost] }` |
| TCP services without HTTP | `tcpSocket` on the service port | `tcpSocket: { port: 6379 }` |

### 5.3 Guidelines

- Prefer `startupProbe` over `initialDelaySeconds` — startup probes adapt to actual initialization time rather than relying on a fixed delay
- Set `failureThreshold` on startup probes high enough to accommodate first-time initialization (database schema creation, file unpacking, etc.)
- Liveness probes should check that the process itself is alive, not that downstream dependencies are healthy — otherwise a database outage cascades into unnecessary pod restarts
- Readiness probes should verify the pod can serve requests end-to-end

---

## 6. Image Security

### 6.1 Image Sources

- Prefer images from trusted registries (e.g., `ghcr.io/<your-org>/*` or your own private registry)
- Mirror public Docker Hub images to a private registry before relying on them in production
- Pin images by tag or SHA digest — avoid `:latest` in production overlays so deploys are reproducible

### 6.2 Image Build

- Use multi-stage builds to keep build tools, source code, and `node_modules` out of the final image
- Base on minimal distributions (Alpine, distroless) where possible
- Keep package managers (`apk`, `apt`), shells, and debugging tools out of the final stage unless the workload genuinely needs them

---

## 7. Network Segmentation & Ingress

### 7.1 Default Deny Posture

Every namespace with production workloads should have NetworkPolicies that collectively cover all pods. A pod without any matching NetworkPolicy has unrestricted ingress and egress.

Declare both `Ingress` and `Egress` in `policyTypes` on every NetworkPolicy. Omitting one leaves that direction unrestricted:

```yaml
policyTypes:
  - Ingress
  - Egress
```

### 7.2 Microsegmentation

Give every distinct workload component its own NetworkPolicy that restricts traffic to the minimum required paths. Follow least privilege — allow only the exact ports, protocols, and source/destination pods the component actually needs.

**Recommended policies per namespace:**

| Policy scope | Ingress allows | Egress allows |
|---|---|---|
| **Databases** (MariaDB, PostgreSQL) | Only application pods on the database port | DNS only |
| **Caches / Queues** (Valkey, Redis) | Only application pods on the cache port | DNS only |
| **Reverse proxies** (Nginx) | Only the ingress controller namespace on the proxy port | Upstream application pods on their service ports, DNS |
| **Application pods** (web servers, workers, schedulers) | Only from the reverse proxy on application ports | Database, cache, DNS, and scoped external egress |

### 7.3 Ingress Rules

Use `podSelector` (same namespace) or `podSelector` + `namespaceSelector` (cross-namespace) to identify allowed sources. Don't leave ingress open to all pods.

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            app.kubernetes.io/app: <app-label>
    ports:
      - protocol: TCP
        port: <service-port>
```

**Cross-namespace ingress** (e.g., ingress controller to application namespace):

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: <ingress-namespace>
    ports:
      - protocol: TCP
        port: <proxy-port>
```

A bare `podSelector` without `namespaceSelector` restricts to the same namespace only — pods from other namespaces are implicitly denied.

### 7.4 Egress Rules

#### Internal egress

Use `podSelector` to scope egress to specific in-namespace services:

```yaml
egress:
  - to:
      - podSelector:
          matchLabels:
            app: <target-service>
    ports:
      - protocol: TCP
        port: <target-port>
```

#### DNS resolution

Any NetworkPolicy with egress restrictions needs a DNS rule — otherwise the pod can't resolve service names.

Scope the `namespaceSelector` to the namespace where CoreDNS/kube-dns actually runs (typically `kube-system`). Using an empty `namespaceSelector: {}` allows egress to any pod labeled `k8s-app: kube-dns` in any namespace — an attacker who can create pods in another namespace could spoof a DNS pod and intercept queries.

Before writing DNS egress rules, confirm CoreDNS pods carry the expected label:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

If no pods are returned, the DNS pods in your cluster use a different label. Find the actual label with `kubectl get pods -n kube-system -o wide | grep -i dns` and update the `podSelector` in all NetworkPolicies accordingly.

```yaml
egress:
  - to:
      - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
        podSelector:
          matchLabels:
            k8s-app: kube-dns
    ports:
      - protocol: UDP
        port: 53
      - protocol: TCP
        port: 53
```

#### External egress

Scope external egress (outside the cluster) to specific ports and exclude RFC 1918 private ranges to prevent lateral movement to other internal services:

```yaml
egress:
  - to:
      - ipBlock:
          cidr: 0.0.0.0/0
          except:
            - 10.0.0.0/8
            - 172.16.0.0/12
            - 192.168.0.0/16
    ports:
      - protocol: TCP
        port: 443     # HTTPS
```

**Common external egress ports:**

| Port | Protocol | When needed |
|------|----------|-------------|
| 443 | TCP | HTTPS — webhooks, API integrations, payment gateways |
| 587 | TCP | SMTP (STARTTLS) — outbound email |
| 465 | TCP | SMTP (implicit TLS) — outbound email |

Only add ports the workload actually requires. Never allow unrestricted egress (`0.0.0.0/0` without port restrictions).

### 7.5 Label Selectors

NetworkPolicies rely on pod labels for targeting. Prefer stable, well-known labels:

| Selector type | Recommended labels |
|---|---|
| Match a specific component | `app.kubernetes.io/name: <component>` |
| Match all components of an app | `app.kubernetes.io/app: <app>` |
| Match multiple components | `matchExpressions` with `operator: In` |
| Exclude a specific component | `matchExpressions` with `operator: NotIn` |

Example — target multiple components:

```yaml
podSelector:
  matchExpressions:
    - key: app.kubernetes.io/name
      operator: In
      values:
        - valkey-cache
        - valkey-queue
```

Example — target all app pods except the reverse proxy:

```yaml
podSelector:
  matchExpressions:
    - key: app.kubernetes.io/app
      operator: In
      values: [frappe]
    - key: app.kubernetes.io/name
      operator: NotIn
      values: [erpnext-nginx]
```

### 7.6 Ingress Controller & TLS

- Use `ClusterIP` Services (not `NodePort` or `LoadBalancer`) unless external access is genuinely needed
- Route external access through the ingress controller (Traefik) with TLS termination
- Apply security headers via Traefik middleware (XSS filter, HSTS, nosniff, rate limiting, `Server` header stripping)

### 7.7 Rate Limiting & IP Spoofing Prevention

Rate limiting middleware should identify clients by their real TCP source IP, not by the `X-Forwarded-For` header — an attacker can rotate through fake `X-Forwarded-For` values per request to bypass per-IP limits entirely.

**Vulnerable configuration — don't use:**

```yaml
sourceCriterion:
  ipStrategy:
    depth: 1    # Trusts X-Forwarded-For — attacker-controlled
```

With this configuration, an attacker can send arbitrary `X-Forwarded-For` values with each request and the rate limiter will treat every request as a different client — rendering the limit ineffective.

**Recommended configuration:**

```yaml
rateLimit:
  average: 10
  burst: 20
  period: 1m
  sourceCriterion:
    requestHost: false
```

Setting `sourceCriterion.requestHost: false` with no `ipStrategy` or `requestHeaderName` makes Traefik v3 use the TCP remote address (connection source IP) as the rate-limit key. It doesn't read or trust `X-Forwarded-For`, so it can't be spoofed at the application layer.

> **Traefik v3 behavior to watch for:**
> - **Omitting `sourceCriterion` entirely** defaults to `requestHost: true`, which groups all requests by hostname — creating a single global bucket per domain instead of per-IP. Common pitfall.
> - **`ipStrategy.depth: 0`** reads the first IP from the `X-Forwarded-For` header, which is attacker-controlled. Don't use this for per-IP rate limiting.
> - **`requestSourceIP: true`** isn't a valid field in Traefik v3 CRDs and will fail validation on apply.

**Prerequisites:**

Configure the ingress controller (Traefik) with `externalTrafficPolicy: Local` on its LoadBalancer Service. This prevents Kubernetes from performing SNAT on incoming traffic and preserves the real client IP as the TCP source address. Without it, every request appears to originate from an internal pod CIDR IP (e.g., `10.42.0.0`), making per-client rate limiting ineffective.

```yaml
# Traefik Service (reference)
apiVersion: v1
kind: Service
metadata:
  name: traefik
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local    # Preserves client source IP
```

With MetalLB L2 mode, `externalTrafficPolicy: Local` requires a Traefik pod on every node MetalLB may elect as the L2 speaker. The simplest approach is to run one Traefik replica per node using `topologySpreadConstraints` with `whenUnsatisfiable: DoNotSchedule`.

**Per-replica rate limit buckets:** Each Traefik replica maintains its own in-memory rate-limit state. With N replicas, the effective rate limit per client can be up to N× the configured value. Account for this when setting `average` and `burst`.

**HSTS configuration:** Set `stsSeconds` to at least `31536000` (1 year) when `stsPreload` is enabled. The HSTS preload list maintained by browsers requires a minimum max-age of 1 year — shorter submissions are rejected.

### 7.8 Cloudflare Proxy: Extracting Real Client IPs

When Cloudflare proxy (orange cloud) is enabled, all client traffic is routed through Cloudflare's edge network. Traefik sees Cloudflare's IP addresses as the TCP source of every request — not the real visitor IP. This breaks the default rate-limiting configuration in Section 7.7, because all visitors share a handful of Cloudflare IP buckets instead of being rate-limited individually.

In this scenario, reading the real client IP from `X-Forwarded-For` becomes **necessary** — but it's only safe when all three of the following controls are in place:

**1. Trust `X-Forwarded-For` only from Cloudflare IPs**

Configure `forwardedHeaders.trustedIPs` on the Traefik entrypoint (Helm `values.yaml`) with Cloudflare's published IP ranges. Traefik will then extract the real client IP from `X-Forwarded-For` only when the connection originates from a Cloudflare IP. Connections from any other IP continue to use the TCP source IP.

```yaml
ports:
  websecure:
    port: 443
    expose:
      default: true
    forwardedHeaders:
      trustedIPs:
        # Cloudflare IPv4 — https://www.cloudflare.com/ips-v4/
        - "173.245.48.0/20"
        - "103.21.244.0/22"
        - "103.22.200.0/22"
        - "103.31.4.0/22"
        - "141.101.64.0/18"
        - "108.162.192.0/18"
        - "190.93.240.0/20"
        - "188.114.96.0/20"
        - "197.234.240.0/22"
        - "198.41.128.0/17"
        - "162.158.0.0/15"
        - "104.16.0.0/13"
        - "104.24.0.0/14"
        - "172.64.0.0/13"
        # Cloudflare IPv6 — https://www.cloudflare.com/ips-v6/
        - "2400:cb00::/32"
        - "2606:4700::/32"
        - "2803:f800::/32"
        - "2405:b500::/32"
        - "2405:8100::/32"
        - "2a06:98c0::/29"
        - "2c0f:f248::/32"
```

**2. Update rate limit middleware to key on `CF-Connecting-IP`**

Set `sourceCriterion.requestHeaderName: Cf-Connecting-Ip` in the rate-limit middleware. Cloudflare sets this header to the single real client IP on every proxied request.

```yaml
sourceCriterion:
  requestHeaderName: Cf-Connecting-Ip
```

> Do **not** use `ipStrategy.depth: 1` behind Cloudflare — the depth strategy can resolve to a rotating Cloudflare edge IP rather than the client, scattering a single client across rate-limit buckets so the limit never trips. This also requires `externalTrafficPolicy: Local` on the Traefik Service (the chart 39.x ignores the value, so patch it: `kubectl patch svc traefik -n <ns> -p '{"spec":{"externalTrafficPolicy":"Local"}}'`).

**3. Restrict origin traffic to Cloudflare only**

Deploy an `IPAllowList` middleware that rejects any request not originating from Cloudflare's IP ranges. This stops attackers from bypassing Cloudflare by connecting directly to the origin and skipping Cloudflare's WAF and DDoS protection. The `cloudflare-only` middleware should be the **first** middleware in every ingress route so non-Cloudflare traffic is rejected before any other processing.

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: cloudflare-only
  namespace: wordpress
spec:
  ipAllowList:
    sourceRange:
      # Same Cloudflare IPv4 and IPv6 ranges as trustedIPs above
      - "173.245.48.0/20"
      - "103.21.244.0/22"
      - "103.22.200.0/22"
      - "103.31.4.0/22"
      - "141.101.64.0/18"
      - "108.162.192.0/18"
      - "190.93.240.0/20"
      - "188.114.96.0/20"
      - "197.234.240.0/22"
      - "198.41.128.0/17"
      - "162.158.0.0/15"
      - "104.16.0.0/13"
      - "104.24.0.0/14"
      - "172.64.0.0/13"
      - "2400:cb00::/32"
      - "2606:4700::/32"
      - "2803:f800::/32"
      - "2405:b500::/32"
      - "2405:8100::/32"
      - "2a06:98c0::/29"
      - "2c0f:f248::/32"
```

**Why this is safe but Section 7.7's default isn't:** In the default setup (no Cloudflare proxy), clients connect directly to Traefik. An attacker can set a fake `CF-Connecting-IP`/`X-Forwarded-For` header and Traefik would trust it — allowing rate-limit bypass. With Cloudflare proxy, control 1 makes Traefik only trust forwarded headers from Cloudflare's IP ranges, Cloudflare sets `CF-Connecting-IP` to the actual connecting client IP, and control 3 guarantees that non-Cloudflare traffic never reaches Traefik.

**IP range maintenance:** Cloudflare publishes its IP ranges at https://www.cloudflare.com/ips-v4/ and https://www.cloudflare.com/ips-v6/. These ranges change infrequently but do change. Three locations need to stay in sync: `forwardedHeaders.trustedIPs` (Helm values), `ipAllowList.sourceRange` (every `cloudflare-only` middleware), and any host or upstream firewall allowlists. A range in the allowlist but missing from `trustedIPs` will allow traffic through but rate-limit it by Cloudflare's IP. A range in `trustedIPs` but missing from the allowlist will reject legitimate traffic with 403.

#### 7.8.1 Automated IP Range Sync

A Kubernetes CronJob (`cloudflare-ip-sync.yaml`) can automate the middleware side of IP range maintenance. A reasonable schedule is weekly (e.g., Monday 06:00 UTC); the job:

1. Fetches the current IPv4 and IPv6 ranges from Cloudflare's published endpoints
2. Discovers every middleware named `cloudflare-only` across all namespaces
3. Compares the fetched ranges against each middleware's `ipAllowList.sourceRange`
4. Patches only the middlewares where drift is detected

The CronJob uses a ClusterRole scoped to `get`, `list`, and `patch` on `middlewares` (Traefik CRD), deployed in the `traefik` namespace alongside the ingress controller so it operates across all namespaces without per-namespace RBAC. It follows the same pod-hardening defaults as application workloads — non-root (UID 65534), read-only root filesystem, all capabilities dropped, seccomp RuntimeDefault.

**Limitation:** The CronJob only updates `IPAllowList` middlewares. The Traefik Helm values (`forwardedHeaders.trustedIPs`) still need to be updated manually via `helm upgrade` — the job logs a warning when drift is detected to surface this. The three sync locations listed above (Helm values, middleware ranges, upstream firewalls) need to be kept aligned by whatever process you adopt.

Apply the CronJob:

```
kubectl apply -f cloudflare-ip-sync.yaml
```

Verify the CronJob and trigger a manual run to confirm it works:

```
kubectl get cronjob cloudflare-ip-sync -n traefik
kubectl create job --from=cronjob/cloudflare-ip-sync cloudflare-ip-sync-manual -n traefik
kubectl logs -n traefik job/cloudflare-ip-sync-manual --follow
```

Configuring the Cloudflare side of the proxy itself (DNS records, origin lockdown rules, and the initial Helm `trustedIPs`) is environment-specific and outside the scope of this guide — complete that setup before relying on this sync job.

### 7.9 Verification

After applying NetworkPolicies, verify segmentation with connectivity tests:

```bash
# List all policies in the namespace
kubectl get networkpolicy -n <namespace>

# Test allowed path (should succeed)
kubectl exec -n <namespace> deploy/<source> -- \
  bash -c "echo > /dev/tcp/<target-service>/<port> && echo 'OK' || echo 'BLOCKED'"

# Test blocked path (should fail)
kubectl exec -n <namespace> deploy/<unrelated-pod> -- \
  sh -c "echo > /dev/tcp/<target-service>/<port> && echo 'OK' || echo 'BLOCKED'" 2>&1
```

---

## 8. Pre-deployment Hardening Checklist

A quick checklist to run through before shipping a workload:

| # | Control | Where to check |
|---|---------|---------------|
| 1 | Custom ServiceAccount (not `default`) | `spec.serviceAccountName` |
| 2 | API token not mounted | `automountServiceAccountToken: false` on SA + Pod |
| 3 | `runAsNonRoot: true` | `spec.securityContext` |
| 4 | `runAsUser` matches Dockerfile UID | `spec.securityContext.runAsUser` |
| 5 | `privileged: false` | `containers[].securityContext` |
| 6 | `allowPrivilegeEscalation: false` | `containers[].securityContext` |
| 7 | `readOnlyRootFilesystem: true` | `containers[].securityContext` |
| 8 | `capabilities.drop: [ALL]` | `containers[].securityContext` |
| 9 | `seccompProfile: RuntimeDefault` | `spec.securityContext` |
| 10 | CPU/memory requests and limits set | `containers[].resources` |
| 11 | Startup, liveness, and readiness probes defined | `containers[].startupProbe`, `livenessProbe`, `readinessProbe` |
| 12 | No `:latest` tag in production | `containers[].image` |
| 13 | Writable paths scoped via `emptyDir` | `volumeMounts` + `volumes` |
| 14 | No `hostPath` volumes | `volumes` |
| 15 | NetworkPolicy with both `Ingress` and `Egress` policyTypes | `NetworkPolicy.spec.policyTypes` |
| 16 | Ingress restricted to required source pods only | `NetworkPolicy.spec.ingress[].from` |
| 17 | Egress restricted to required destinations + DNS | `NetworkPolicy.spec.egress[].to` |
| 18 | External egress excludes RFC 1918 ranges | `ipBlock.except` includes `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` |
| 19 | No unrestricted egress (no `0.0.0.0/0` without port scoping) | `NetworkPolicy.spec.egress[].ports` |
| 20 | Rate limiting uses `requestHost: false` with no `ipStrategy` (unless Cloudflare proxy — see Section 7.8) | `sourceCriterion` in Traefik middleware |
| 21 | Traefik Service uses `externalTrafficPolicy: Local` | `kubectl get svc traefik -n traefik` |
