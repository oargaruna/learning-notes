# Pod Security Admission (PSA)

Notes on PSA — the built-in admission controller that enforces the Pod Security Standards on pod specs. Covers the overall model, one specific rule addition (probe `.host` block), and a concrete rollout scenario on AKS.

## Contents

- [summary.md](./summary.md) — what PSA is, the three PSS levels, the three modes, where it sits in the admission chain, and what it deliberately doesn't cover.
- [host-field-restriction.md](./host-field-restriction.md) — KEP-4940: why `.host` on probes and lifecycle handlers is a blind-SSRF primitive via kubelet, and how Baseline@v1.34 blocks it.
- [aks-usage-scenario.md](./aks-usage-scenario.md) — end-to-end rollout on a multi-tenant AKS cluster: enable audit logs, label namespaces audit+warn, query violations in Log Analytics, fix, flip to enforce.
