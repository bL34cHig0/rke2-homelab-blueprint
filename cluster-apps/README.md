# Kubernetes Application Manifests

Helm values and raw manifests for the cluster-level platform pieces and a reference workload. Each subdirectory has its own `README.md` with installation, upgrade, and troubleshooting steps.

| Directory | Role | Highlights |
|---|---|---|
| [`longhorn/`](./longhorn/) | Distributed block storage | Install via the Rancher Apps & Marketplace UI, multipathd prerequisite, LUKS-encrypted `StorageClass` (global key) wired to a `longhorn-crypto` Secret, per-volume key pattern, verification and troubleshooting notes |
| [`traefik/`](./traefik/) | Ingress + TLS termination | Traefik v3 Helm values, cert-manager (Let's Encrypt DNS-01 via Cloudflare), wildcard `Certificate` for `*.<your-domain>`, dashboard `IngressRoute`, default security headers, rate-limit middleware, troubleshooting notes |
| [`rancher/`](./rancher/) | Cluster management UI | Rancher Helm install with `tls=external` (Traefik terminates TLS), wildcard certificate hand-off, default security headers, ingress route |
| [`prometheus-grafana/`](./prometheus-grafana/) | Observability | kube-prometheus-stack values, custom `PrometheusRule` set, Alertmanager → Discord webhook, dedicated storage class |
| [`wordpress/`](./wordpress/) | Reference workload | End-to-end pattern (Deployment, Service, ServiceAccount, ConfigMap, Secret, PVC, Certificate, IngressRoute, default-headers + deny-endpoints middleware, hardened MariaDB) that future workloads can be modelled on |

## Table of Contents

- [Install Order](#install-order)
- [Before Applying Anything](#before-applying-anything)

## Install Order

Within this directory the dependency order is:

1. `traefik/cert-manager/` — install cert-manager, create the Cloudflare API-token secret, then the `ClusterIssuer` (staging first, then production), then the wildcard `Certificate`.
2. `traefik/` — install Traefik with the values file, then apply `default-headers.yaml` and the `dashboard/` ingress.
3. `rancher/` — apply the Rancher `Certificate`, install Rancher with `tls=external`, then apply ingress + headers.
4. `longhorn/` — disable `multipathd` on all nodes, install Longhorn via the Rancher Apps & Marketplace UI, then (optionally) create the `longhorn-crypto` Secret and apply the encrypted `StorageClass`. Provides the storage layer the next two apps' PVCs bind to.
5. `prometheus-grafana/` — create the storage class, install kube-prometheus-stack with the supplied values, then apply `prometheus-rules.yaml` and the Discord webhook secret.
6. `wordpress/` — apply manifests in the order documented in `wordpress/README.md` (MariaDB ConfigMap/Secret → ServiceAccount → PVC → Deployment → Service → WordPress equivalents → Certificate → Middleware → IngressRoute).

## Before Applying Anything

Replace every placeholder (`<your-domain>`, `<your-email>`, `<your-org>`, `<your-namespace>`) and adjust the `default-cert-production` / `default-cert-staging` certificate and secret names to match your naming convention. See the root `README.md` for the full placeholder table.
