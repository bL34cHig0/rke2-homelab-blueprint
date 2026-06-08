# Traefik Rate Limiting Troubleshooting

This document covers the requirements, common issues, and troubleshooting steps for per-IP rate limiting with Traefik v3 on RKE2 with MetalLB L2 mode.

## Table of Contents

- [How Per-IP Rate Limiting Works](#how-per-ip-rate-limiting-works)
- [Requirements](#requirements)
- [Common Issues](#common-issues)
  - [All users share a single rate limit bucket](#all-users-share-a-single-rate-limit-bucket)
  - [externalTrafficPolicy: Local causes site to be unreachable](#externaltrafficpolicy-local-causes-site-to-be-unreachable)
  - [Helm upgrade does not apply externalTrafficPolicy](#helm-upgrade-does-not-apply-externaltrafficpolicy)
  - [Access logs show pod CIDR IP for all requests](#access-logs-show-pod-cidr-ip-for-all-requests)
  - [Rate limit appears more permissive than configured](#rate-limit-appears-more-permissive-than-configured)
- [Diagnostic Commands](#diagnostic-commands)
- [Traefik v3 sourceCriterion Behavior](#traefik-v3-sourcecriterion-behavior)

---

## How Per-IP Rate Limiting Works

For rate limiting to work per-IP, Traefik must see the real client IP as the TCP connection source. This requires the following chain:

```
Client (real IP) → MetalLB L2 (ARP) → Node (no SNAT) → Traefik pod (sees real IP)
```

With `externalTrafficPolicy: Cluster` (the default), kube-proxy performs Source NAT (SNAT) on incoming traffic, replacing the client IP with an internal pod CIDR address (e.g., `10.42.0.0`). Traefik sees this internal IP for every request, making all clients share a single rate limit bucket.

With `externalTrafficPolicy: Local`, kube-proxy skips SNAT and delivers traffic directly to the Traefik pod on the receiving node. The real client IP is preserved.

---

## Requirements

All three requirements must be met for per-IP rate limiting to function:

### 1. externalTrafficPolicy: Local

The Traefik LoadBalancer Service must use `externalTrafficPolicy: Local`:

```yaml
# Traefik Helm values.yaml
service:
  externalTrafficPolicy: Local
```

Verify:

```bash
kubectl get svc traefik -n traefik -o jsonpath='{.spec.externalTrafficPolicy}'
# Expected: Local
```

### 2. One Traefik pod per node

With `externalTrafficPolicy: Local`, traffic arriving at a node is only delivered to a local Traefik pod. If no Traefik pod exists on the node that MetalLB elected as the L2 speaker, traffic has nowhere to go and the site becomes unreachable.

The solution is to run one Traefik replica per node:

```yaml
# Traefik Helm values.yaml
deployment:
  replicas: 4    # Match your total node count (control plane + workers)
  podSpec:
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: "kubernetes.io/hostname"
        whenUnsatisfiable: DoNotSchedule    # Strict: one pod per node
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: traefik
    tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
```

The toleration is required so a Traefik pod can schedule on the control plane node. Without it, one node has no Traefik pod and `Local` policy fails if MetalLB elects that node.

Verify one pod per node:

```bash
kubectl get pods -n traefik -l app.kubernetes.io/name=traefik -o wide
```

### 3. Correct sourceCriterion

The rate limit middleware must use `requestHost: false` with no `ipStrategy`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: ratelimit
spec:
  rateLimit:
    average: 10
    burst: 20
    period: 1m
    sourceCriterion:
      requestHost: false
```

This tells Traefik to use the TCP remote address as the rate limit key.

---

## Common Issues

### All users share a single rate limit bucket

**Symptom:** When one user triggers the rate limit, other users from different IPs also receive `429 Too Many Requests`.

**Possible causes:**

| Cause | How to identify | Fix |
|-------|----------------|-----|
| `externalTrafficPolicy: Cluster` | `kubectl get svc traefik -n traefik -o jsonpath='{.spec.externalTrafficPolicy}'` returns `Cluster` | Patch to `Local` (see below) |
| `sourceCriterion` omitted entirely | Check middleware YAML — no `sourceCriterion` block | Traefik v3 defaults to `requestHost: true` (groups by hostname, not IP). Add `sourceCriterion.requestHost: false` |
| `sourceCriterion.requestHost: true` | Explicitly set in middleware | All requests to the same domain share one bucket. Set to `false` |
| `ipStrategy.depth: 0` | Set in `sourceCriterion` | In Traefik v3, `depth: 0` reads the first `X-Forwarded-For` value (spoofable). Remove `ipStrategy` and use `requestHost: false` |

### externalTrafficPolicy: Local causes site to be unreachable

**Symptom:** After patching the Traefik service to `Local`, the site returns DNS errors or connection timeouts.

**Cause:** MetalLB L2 mode elects one node as the ARP responder for the VIP. With `Local` policy, traffic arriving at that node is only delivered to a local Traefik pod. If no Traefik pod exists on the elected node, traffic is dropped.

**Fix:** Ensure every node has a Traefik pod. Increase replicas to match your node count and use `DoNotSchedule` topology spread constraints. See [Requirements](#2-one-traefik-pod-per-node).

**Verification:**

```bash
# Check which nodes have Traefik pods
kubectl get pods -n traefik -l app.kubernetes.io/name=traefik -o wide

# Check all nodes in the cluster
kubectl get nodes

# Every node should have a Traefik pod. If not, check:
# - Is replicas >= node count?
# - Is the control plane toleration present?
# - Is topologySpreadConstraints set to DoNotSchedule?
```

### Helm upgrade does not apply externalTrafficPolicy

**Symptom:** The Traefik Helm values.yaml has `externalTrafficPolicy: Local` but the live service shows `Cluster` after `helm upgrade` (or after a fresh install).

**Cause:** Two distinct failure modes:
1. **The chart never templates the value (chart v36+ / 39.x).** `service.externalTrafficPolicy` is accepted in values but is **not** rendered into the Service at all — so the Service is born `Cluster` even on a clean install. Verify with `helm get manifest traefik -n traefik | grep -i externaltraffic` (returns nothing) while `helm get values traefik -n traefik` shows your `Local`. A patch (below) is the reliable, version-independent fix.
2. **Helm won't update an existing Service's field.** Even when rendered, Helm's three-way merge may not update an already-set `externalTrafficPolicy` on a live Service.

Either way the symptom is the same: `Cluster`, SNAT, no real client IP.

**Fix:** Patch the service manually after the Helm upgrade:

```bash
kubectl patch svc traefik -n traefik -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

Verify:

```bash
kubectl get svc traefik -n traefik -o jsonpath='{.spec.externalTrafficPolicy}'
# Expected: Local
```

**Note:** This manual patch may be reverted by a subsequent `helm upgrade`. If this happens, add the patch as a post-upgrade step in your deployment process, or use a Helm post-upgrade hook.

### Access logs show pod CIDR IP for all requests

**Symptom:** Traefik access logs show `10.42.0.0` (or similar pod CIDR) as the client IP for all external requests.

**Cause:** `externalTrafficPolicy: Cluster` is in effect — kube-proxy is performing SNAT.

**Important:** Internal cluster traffic (pod-to-pod, Rancher dashboard, Kubernetes API) will always show internal IPs regardless of the traffic policy. Only external requests (from outside the cluster) should show real client IPs.

**How to distinguish:**

```bash
# Enable access logging in Traefik values.yaml
logs:
  access:
    enabled: true

# After upgrading, make an external request
curl -s https://your-domain.com/ > /dev/null

# Check logs — filter for your application's IngressRoute
kubectl logs -n traefik -l app.kubernetes.io/name=traefik --tail=20 | grep "your-namespace"
```

External requests should show your public IP. Internal requests (Rancher, cert-manager, etc.) will continue showing `10.42.x.x` — this is expected.

### Rate limit appears more permissive than configured

**Symptom:** Configured `average: 10` with `period: 1m` but a client can make 30-40 requests before being throttled.

**Cause:** Each Traefik replica maintains its own independent in-memory rate limit bucket. With 4 replicas, MetalLB and kube-proxy distribute requests across pods, so a single client's requests are spread across up to 4 separate buckets.

**Effective rate limit:** Up to `configured_value * replica_count`. With `average: 10` and 4 replicas, the effective limit is up to 40 requests per period.

**Mitigation:** Divide the desired rate limit by the replica count. For example, to achieve an effective limit of ~10 req/min with 4 replicas, set `average: 3`.

This is a known Traefik limitation — the rate limiter is in-memory and per-instance. There is no built-in mechanism to share rate limit state across replicas. For stricter rate limiting, consider:
- Reducing `average` and `burst` values to compensate
- Using Cloudflare's WAF rate limiting at the edge (see `cloudflare-proxy.md`)
- Adding a WordPress login protection plugin as defense in depth

---

## Diagnostic Commands

```bash
# 1. Check externalTrafficPolicy
kubectl get svc traefik -n traefik -o jsonpath='{.spec.externalTrafficPolicy}'

# 2. Check Traefik pod distribution (one per node)
kubectl get pods -n traefik -l app.kubernetes.io/name=traefik -o wide

# 3. Check rate limit middleware configuration
kubectl get middleware ratelimit -n wordpress -o yaml

# 4. Enable access logging (Traefik values.yaml)
# Add: logs.access.enabled: true
# Then: helm upgrade traefik traefik/traefik -n traefik -f values.yaml

# 5. Check access logs for client IPs (external traffic)
kubectl logs -n traefik -l app.kubernetes.io/name=traefik --tail=30 | grep "wordpress"

# 6. Check MetalLB speaker status
kubectl get pods -n metallb-system -l component=speaker -o wide

# 7. Check MetalLB L2 advertisement configuration
kubectl get l2advertisement -n metallb-system -o yaml

# 8. Test rate limiting from an external IP
# Run rapidly from one IP — should eventually get 429
curl -s -o /dev/null -w "%{http_code}" https://your-domain.com/wp-login.php

# 9. Verify another IP is unaffected
# From a different network/IP — should still get 200
curl -s -o /dev/null -w "%{http_code}" https://your-domain.com/wp-login.php
```

---

## Traefik v3 sourceCriterion Behavior

The `sourceCriterion` field in Traefik v3 rate limit middleware has three mutually exclusive sub-fields. Only one should be set — if multiple are set, the priority order is: `ipStrategy` > `requestHeaderName` > `requestHost`.

| Field | Behavior | Use case |
|-------|----------|----------|
| `requestHost: true` | Groups all requests by the `Host` header value. All visitors to the same domain share one bucket. | **Default when `sourceCriterion` is omitted.** Almost never what you want for per-client rate limiting. |
| `requestHost: false` | Uses the TCP remote address (direct connection IP) as the rate limit key. | **Recommended for direct traffic** (no reverse proxy in front of Traefik). Requires `externalTrafficPolicy: Local` to see real client IPs. |
| `ipStrategy.depth: N` | Reads the Nth IP from the `X-Forwarded-For` header (0 = first/leftmost). | Generic trusted reverse proxies. **Avoid behind Cloudflare** — depth can resolve to a *rotating* CF edge IP and scatter one client across buckets so the limit never trips; use `requestHeaderName: Cf-Connecting-Ip` instead. Must be combined with `forwardedHeaders.trustedIPs`. |
| `requestHeaderName: "Header-Name"` | Groups requests by the value of the specified header. | **Recommended behind Cloudflare:** `requestHeaderName: Cf-Connecting-Ip` — the single real client IP Cloudflare sets on every request. Also API-key-style limiting. |

### Common pitfalls

| Configuration | Expected behavior | Actual behavior in Traefik v3 |
|--------------|-------------------|-------------------------------|
| `sourceCriterion` omitted | Per-IP rate limiting | **Global per-hostname** — all clients share one bucket |
| `ipStrategy.depth: 0` | Use TCP remote address | **Reads first `X-Forwarded-For` value** — spoofable |
| `requestSourceIP: true` | Per-IP rate limiting | **Invalid field** — rejected by CRD validation |
| `requestHost: false` (no other fields) | Use TCP remote address | **Correct** — uses TCP remote address |
