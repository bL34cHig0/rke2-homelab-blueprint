# Contributing

Thanks for your interest in improving this blueprint. Contributions of all sizes are welcome — hardening fixes, simpler manifests, better defaults, additional reference workloads, security review, or clearer documentation.

## Ways to Contribute

- **Documentation** — fix errors, clarify steps, improve explanations, add diagrams.
- **Hardened manifests for other cluster apps** — GitOps controllers (Argo CD, Flux), auth providers (Authentik, Keycloak, Dex), secret management (External Secrets Operator, Vault, Sealed Secrets), backups (Velero), networking (Cilium, Istio, Linkerd), observability (Loki, Tempo, Thanos, OpenTelemetry), or popular self-hosted apps (Vaultwarden, Nextcloud, Immich, Home Assistant, GitLab).
- **Infrastructure-as-code automation** — Terraform modules for the host layer (VMs, networks, firewalls) and Ansible playbooks to automate OS prep and RKE2 installation, replacing the currently manual `infrastructure/` steps.
- **Security review** — flag misconfigurations, weak defaults, or missing controls.

## Before You Open a PR

1. **Open an issue first** for anything non-trivial so the approach can be discussed before you invest time.
2. **Keep changes focused** — one logical change per PR makes review faster.
3. **Never commit secrets.** All credentials in this repo are placeholders or template Secrets created imperatively. Do not add real domains, emails, IPs, tokens, kubeconfigs, or `.env` files. See [SECURITY.md](SECURITY.md).
4. **Replace identifiers with placeholders** — use `<your-domain>`, `<your-email>`, `<your-org>`, `<your-namespace>` (see the [Placeholder Convention](README.md#placeholder-convention)).
5. **Redact screenshots** — blur public IPs, domains, emails, fingerprints, and any account identifiers, matching the existing `images/` convention. Add descriptive `alt` text for every image.

## Style Conventions

- **Commits** follow [Conventional Commits](https://www.conventionalcommits.org/) — e.g. `docs(longhorn): add encrypted StorageClass`, `fix(wordpress): ...`.
- **Markdown docs** start with an `## Table of Contents` for multi-section files; pin external doc links to a specific version where the upstream offers versioned docs.
- **Manifests** mirror the existing per-app folder structure and the hardening patterns in [infrastructure/security/k8s-deployment-security-policy.md](infrastructure/security/k8s-deployment-security-policy.md): non-root UIDs, read-only root filesystems, dropped capabilities, dedicated ServiceAccounts with `automountServiceAccountToken: false`, and a NetworkPolicy per workload.

## Reporting Bugs or Suggesting Improvements

Open an [issue](../../issues) describing the problem or idea, the affected file(s), and (for bugs) the steps to reproduce and what you expected. For security-sensitive reports, follow [SECURITY.md](SECURITY.md) instead of opening a public issue.

The goal is to grow this into a broader catalog of production-ready, hardened patterns that future operators can adopt without re-deriving the same security choices. Thanks for helping.
