# PSA summary

PSA is the **built-in validating admission controller** in kube-apiserver that enforces the Pod Security Standards (PSS) on pod specs. GA in v1.25; replaced the deprecated PodSecurityPolicy.

Core idea: **no webhook, no CRDs, no policy language** — the operator picks a level and a mode per namespace via labels. Everything runs in-process.

## The three standards (levels)

- `privileged` — no restrictions; for system workloads that legitimately need host access.
- `baseline` — blocks known privilege-escalation techniques (hostPath, hostNetwork, hostPID/IPC, privileged, hostProcess, NET_RAW added capabilities, and — since v1.34 — `.host` on probes).
- `restricted` — hardened profile: `runAsNonRoot`, `allowPrivilegeEscalation: false`, seccomp `RuntimeDefault`, all capabilities dropped, volume types restricted, etc.

Each standard has a **version** pinned to a Kubernetes release (e.g. `v1.34`). New checks are added as part of a new version — older pinned versions keep their old rule set.

## The three modes

Set independently per namespace via labels. A namespace can use different levels for different modes (e.g. enforce at Baseline, audit at Restricted).

| Label | On violation |
|---|---|
| `pod-security.kubernetes.io/enforce` | Reject the request at admission |
| `pod-security.kubernetes.io/audit` | Allow, annotate the kube-apiserver audit event |
| `pod-security.kubernetes.io/warn` | Allow, return a warning to the client (visible in `kubectl`) |

Each has an optional `-version` suffix (e.g. `enforce-version: v1.34`).

**Scope difference:** `enforce` evaluates **pods only**. `audit` and `warn` also evaluate pod-creating controllers (Deployments, StatefulSets, Jobs, CronJobs, …), which catches issues at the workload-author layer before any ReplicaSet tries to create a rejected pod.

## Where it sits in the admission chain

```
mutating webhooks ─► schema validation ─► built-in validators (incl. PSA) ─► VAP ─► validating webhooks ─► etcd
```

PSA runs in-process, before validating webhooks. If PSA rejects, nothing downstream runs — so a webhook can further restrict but can't loosen PSA.

## Configuration surface

- **Namespace labels** — primary knob; each namespace picks level + mode + version.
- **`AdmissionConfiguration`** file on kube-apiserver (`--admission-control-config-file`) — cluster defaults for unlabeled namespaces, plus **exemptions** by username, runtime class, or namespace. Not user-editable on most managed platforms (including AKS).

## What PSA deliberately doesn't cover

PSA is scoped to pod security — not general admission. It can't:

- Enforce rules on non-pod resources (Services, Ingresses, ConfigMaps, CRDs).
- Reference external state (an allowlist CRD, an approved-registry list).
- Look at fields other than the PSS-relevant subset (e.g. the `image` field or resource names).

For anything beyond pod security, use dynamic admission: `ValidatingAdmissionPolicy` (in-tree CEL, GA v1.30) for pure object-shape rules, or a policy engine (Kyverno, Gatekeeper) for richer logic.

## Observability

- **Metric:** `apiserver_pod_security_evaluations_total{decision, mode, level}` — surfaces deny counts per mode and level.
- **Audit annotation:** `pod-security.kubernetes.io/audit-violations` on the apiserver audit event. Only useful if audit logging is actually wired to a sink.
- **Client warning:** `warn` mode surfaces inline in `kubectl apply` output.

## Rollout pattern

Canonical sequence for tightening a namespace:

1. Label audit + warn at the target level and version. Enforce stays permissive.
2. Watch the audit log and deny metrics. Fix offending workloads.
3. Flip enforce to the target level once clean.
4. Keep audit pinned one notch stricter than enforce (e.g. enforce=baseline, audit=restricted) to surface future hardening work.

Version pinning is the rollback lever. If a new PSA version introduces a rule that breaks you, pin the namespace's `enforce-version` to the prior release and upgrade on your own timeline.
