# Security Policy

This repository is a **documentation and configuration blueprint** — a set of Kubernetes manifests, Helm values, and operational guides. It is not a deployed service. "Vulnerabilities" here generally mean **insecure defaults, weak configurations, or guidance that would lead an operator to a less secure outcome**, rather than exploitable bugs in running software.

## Scope

In scope:

- Insecure or misleading configuration defaults in the manifests, Helm values, or guides.
- Hardening guidance that is incorrect, outdated, or that weakens the documented security posture.
- Accidentally committed secrets (real tokens, keys, kubeconfigs, credentials, private IPs/domains/emails).

Out of scope:

- Vulnerabilities in the upstream projects themselves (RKE2, Rancher, Longhorn, Traefik, cert-manager, MetalLB, Prometheus, WordPress, etc.) — report those to their respective maintainers.
- Issues that only arise from ignoring the documented hardening steps.

## Reporting a Vulnerability

**Do not open a public issue for sensitive reports.**

Use GitHub's **[Private vulnerability reporting](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability)** for this repository (Security tab → "Report a vulnerability"). If that is unavailable, open a minimal public issue asking for a private contact channel without disclosing details.

When reporting, please include:

- The affected file(s) and line(s).
- A description of the insecure default or weakness.
- The impact (what an attacker could do, or what an operator would get wrong).
- A suggested fix, if you have one.

## Accidentally Committed Secrets

If you find a committed secret (token, key, kubeconfig, real credential, or identifying IP/domain/email), report it privately as above. Note that **rotating the exposed credential is the only real remediation** — removing it from the latest commit does not remove it from git history.

## Response

This is a community project maintained on a best-effort basis. Reports will be acknowledged and addressed as quickly as is practical. Thank you for helping keep the blueprint safe to adopt.
