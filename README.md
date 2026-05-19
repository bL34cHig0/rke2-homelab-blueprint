# RKE2 Cluster Blueprint

A reproducible blueprint for an RKE2-based Kubernetes cluster on Ubuntu hosts. It covers OS preparation, cluster installation, CIS hardening, networking (MetalLB, NAT egress), ingress with Traefik + cert-manager (Let's Encrypt wildcard), and reference deployments for Rancher, WordPress, and a Prometheus/Grafana monitoring stack.

This repository is a starting point — every domain, email, namespace and organization identifier appears as a placeholder you must replace before applying any manifest.

## Table of Contents

- [Background](#background)
- [Placeholder Convention](#placeholder-convention)
- [Layout](#layout)
- [Suggested Order of Operations](#suggested-order-of-operations)
- [Pod Security Baseline](#pod-security-baseline)
- [Suitable Hosting Environments](#suitable-hosting-environments)
- [Acknowledgments](#acknowledgments)
- [Contributing](#contributing)

## Background

This blueprint is the product of roughly 7 months of building and rebuilding Kubernetes clusters from scratch with security as a first-class concern — host hardening, CIS Benchmark conformance, network microsegmentation, image and pod security policy, and the operational habits that surround them. The repo distills those iterations into a coherent starting point others can adopt without repeating every trial-and-error step I went through.

Decisions here reflect specific tradeoffs and learning moments. Not every choice will be optimal for every environment, but each one has a stated reason in the docs — the path forward is to refine them in the open, not to hide them.

## Placeholder Convention

| Placeholder | Replace with |
|---|---|
| `<your-domain>` | The base domain you control (e.g. `example.com`) |
| `<your-email>` | The email used for Let's Encrypt registration and admin contacts |
| `<your-org>` | Your container registry org / GitHub org (e.g. `acme-corp`) |
| `<your-namespace>` | The Kubernetes namespace for the workload being applied |
| `default-cert-production` / `default-cert-staging` | Names of the cert-manager `Certificate` resources and their TLS secrets |

The blueprint defaults to a **3-level hostname layout** (`<service>.<your-domain>`, e.g. `rancher.example.com`). This is the simplest path: Cloudflare's default edge certificate covers it, and a single Let's Encrypt wildcard for `*.<your-domain>` issued via DNS-01 covers every cluster service.

If you prefer to **group cluster services under a dedicated subdomain** (`<service>.<cluster-subdomain>.<your-domain>`, e.g. `rancher.k8s.example.com`), that's a 4th-level hostname. It's a valid choice — it segments cluster traffic from anything else on the root domain — but it triggers a Cloudflare edge-certificate constraint that has its own opt-in appendix in [cluster-apps/traefik/README.md](cluster-apps/traefik/README.md). Read that section before opting in.

## Layout

```
.
├── infrastructure/                  Cluster build & operations docs
│   ├── rke2-ubuntu-prerequisites.md
│   ├── ubuntu-setup-and-user-provisioning.md
│   ├── ssh-access-to-worker-srv-via-ctrl-plane.md
│   ├── rke2-installation.md
│   ├── rke2-cis-self-assessment-benchmark-config.md
│   ├── metallb-load-balancer-config.md
│   ├── nat-gateway-config.md
│   ├── srv-security-and-firewall-config.md
│   ├── node-maintenance.md
│   ├── uninstalling-or-deactivating-auto-configuration-package.md
│   └── security/
│       └── k8s-deployment-security-policy.md
├── images/                          Screenshots referenced by infrastructure guides
└── cluster-apps/                    Helm values + Kubernetes manifests
    ├── traefik/                     Ingress + cert-manager (Let's Encrypt DNS-01)
    ├── rancher/                     Cluster management UI
    ├── prometheus-grafana/          kube-prometheus-stack values, alert rules
    └── wordpress/                   Reference workload (WP + MariaDB)
```

## Suggested Order of Operations

1. **Host preparation** — `infrastructure/rke2-ubuntu-prerequisites.md`, `infrastructure/ubuntu-setup-and-user-provisioning.md`, `infrastructure/ssh-access-to-worker-srv-via-ctrl-plane.md`
2. **Networking** — `infrastructure/nat-gateway-config.md`, `infrastructure/srv-security-and-firewall-config.md`
3. **Cluster install** — `infrastructure/rke2-installation.md`
4. **CIS hardening** — `infrastructure/rke2-cis-self-assessment-benchmark-config.md`
5. **Load balancer** — `infrastructure/metallb-load-balancer-config.md`
6. **Ingress + TLS** — `cluster-apps/traefik/` (cert-manager, then Traefik, then dashboard)
7. **Cluster management** — `cluster-apps/rancher/`
8. **Observability** — `cluster-apps/prometheus-grafana/`
9. **Application workload** — `cluster-apps/wordpress/` (reference pattern for new apps)
10. **Operations** — `infrastructure/node-maintenance.md`, `infrastructure/uninstalling-or-deactivating-auto-configuration-package.md`

## Pod Security Baseline

[`infrastructure/security/k8s-deployment-security-policy.md`](infrastructure/security/k8s-deployment-security-policy.md) collects the workload hardening practices this blueprint follows by default — non-root UIDs, read-only root filesystems, dropped Linux capabilities, dedicated service accounts with `automountServiceAccountToken: false`, network-policy microsegmentation, image-registry allowlisting, and the rate-limit / `X-Forwarded-For` posture for Cloudflare-fronted traffic.

Treat these as a recommended baseline rather than hard requirements. Each control exists for a stated reason in the doc, so you can make an informed call about which ones fit your threat model, which ones to tune, and which ones to relax. The defaults aim for "production-ready out of the box," but a homelab on a private network will reasonably need fewer controls than an internet-facing production cluster.

## Suitable Hosting Environments

This blueprint is sized for a small RKE2 cluster — typically 1 control-plane + 2 workers, or a single-node cluster for testing. It runs on anything that gives you root-shell Ubuntu LTS VMs (or bare metal) with public IPv4 and the ability to load kernel modules. It's well-suited as a homelab or self-hosted staging environment on any of the following cheap/affordable providers:

- **[Hetzner Cloud](https://www.hetzner.com/cloud)** — best price/performance in this class. A 3-node cluster of CX22 instances (2 vCPU, 4 GB RAM) is roughly €15/month total. EU-based, with US locations available.
- **[Hostinger VPS](https://www.hostinger.com/vps-hosting)** — KVM VPS plans start under $5/month per node. Convenient if you're already using Hostinger for DNS or a managed domain.
- **[Contabo](https://contabo.com/)** — generous RAM/CPU at low prices, EU- and US-based. Watch the network egress caps on cheaper tiers.
- **[OVHcloud](https://www.ovhcloud.com/)** — VPS line for small clusters; Eco / SoYouStart bare-metal for heavier workloads.
- **[Scaleway](https://www.scaleway.com/)** — French cloud, comparable to Hetzner. Their `Stardust` nano instances are useful for low-cost dev nodes.
- **[Vultr](https://www.vultr.com/)** / **[DigitalOcean](https://www.digitalocean.com/)** / **[Linode (Akamai)](https://www.linode.com/)** — slightly pricier than the EU options but easy ramps if you're already on those clouds.
- **[Oracle Cloud — Always Free](https://www.oracle.com/cloud/free/)** — up to 4 Arm Ampere cores + 24 GB RAM free forever, when capacity is available (it's frequently constrained). A full free-tier cluster is doable, with the caveat that scheduling new VMs can be unreliable.
- **Bare-metal / home lab** — any spare Intel NUC, Mini PC, or repurposed desktop running Ubuntu LTS works. RKE2 has modest baseline overhead (~1 GB RAM for kube-system per node); plan for at least 3 GB per node to leave room for workloads.

> **What to avoid:** "container-style" or OpenVZ-based budget VPS plans without true kernel access. RKE2 needs to load kernel modules and run an embedded containerd, so stick to KVM/Xen/Hyper-V VPS, dedicated servers, or bare metal. If you're unsure, look for the provider explicitly advertising "KVM VPS."

For sizing, a minimal viable cluster is ~6 GB RAM total (1 GB per node baseline + headroom for Traefik, cert-manager, Rancher, Prometheus, and your apps). A 3-node Hetzner CX22 cluster comfortably runs everything in this blueprint plus a small reference workload like the WordPress example.

## Acknowledgments

Special thanks to **[TechnoTim](https://www.youtube.com/@technotim)**, whose tutorials laid the original groundwork for this project. The practical Kubernetes, self-hosting, and homelab content freely available on his channel is what made it possible to build this from first principles. This repo stands on that foundation and pushes it further into hardened, opinionated defaults.

## Contributing

Contributions are welcome — hardening fixes, simpler manifests, better defaults, additional reference workloads, security review, or clearer documentation. If you spot something that could be more idiomatic, more secure, or just better explained, open a PR or an issue. The goal is for this to be a living blueprint, not a one-off snapshot — best practices on configuring and managing Kubernetes clusters, workloads, and the supporting infrastructure evolve quickly, and the repo should evolve with them.

**Security-hardened manifests for other prominent cluster apps and infrastructure components are especially welcome** — for example, GitOps controllers (Argo CD, Flux), auth providers (Authentik, Keycloak, Dex), secret management (External Secrets Operator, Vault, Sealed Secrets), backups (Velero), networking (Cilium, Istio, Linkerd), observability extensions (Loki, Tempo, Thanos, OpenTelemetry), or popular self-hosted apps (Vaultwarden, Nextcloud, Immich, Home Assistant, GitLab). The aim is to grow this into a broader catalog of production-ready, hardened patterns that future operators can adopt without re-deriving the same security choices.

If you find this useful or beneficial in your own homelab, learning journey, or production setup, please consider **starring the repo** and **sharing it** with others who might get value from it. Visibility is what turns a personal project into a community resource.

**Use this for ethical purposes only.** This blueprint is intended for legitimate self-hosting, learning, research, and production deployments you own or are authorized to operate. Don't use it to stand up infrastructure for fraud, abuse, illegal services, or any workload that harms others.
