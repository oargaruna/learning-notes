# PSA: blocking `.host` in probes and lifecycle handlers (KEP-4940)

Stable in Kubernetes **v1.34**. Added to the **Baseline** PSS. **No feature gate** — the check is gated purely on the PSA version label pinned on the namespace.

## The problem

`HTTPGetAction` and `TCPSocketAction` both expose an optional `.host` field, reachable at these paths on every `Container` and `InitContainer`:

- `...{liveness,readiness,startup}Probe.{httpGet,tcpSocket}.host`
- `...lifecycle.{postStart,preStop}.{httpGet,tcpSocket}.host`

Default (`.host` unset): kubelet probes the **pod's own IP** — the intended behavior.
Override: kubelet probes **whatever string the pod author wrote**, from **the node's network position**.

That turns `.host` into a blind-SSRF primitive via kubelet:

- `127.0.0.1` — services bound on the node (kubelet's own endpoints, node-exporter, cloud-agent sidecars, log shippers).
- `169.254.169.254` — IMDS. On AKS this is how **kubelet itself** fetches AAD tokens for its managed identity at image-pull time. A pod author who can set `.host: 169.254.169.254` can force-probe IMDS. Not direct token theft (response body isn't returned to the pod), but reachability + timing side channel from the kubelet's vantage point.
- Other node IPs, control-plane endpoints, internal admin dashboards — anything routable from the node that may not be reachable from the pod network.

Probe response bodies aren't surfaced to the container, **but success/failure is** — failed probes restart containers, log events, and change pod status. At minimum that's a port scanner originating from the node.

## What KEP-4940 changes

**Baseline** PSS rejects `.host` on every probe and lifecycle handler path listed above when the namespace is configured with:

```yaml
pod-security.kubernetes.io/enforce: baseline   # or restricted
pod-security.kubernetes.io/enforce-version: v1.34
```

Rejection message:

```
pods "liveness-http" is forbidden: violates PodSecurity "baseline:v1.34":
probeHost (container "liveness" uses probeHost 135.45.63.4)
```

**Existing pods** with `.host` set are untouched. The check fires only on pod create/update. Controller-owned pods only get re-evaluated when the controller creates a fresh replica.

## Rollback

Pin the namespace to the previous PSA version:

```yaml
pod-security.kubernetes.io/enforce-version: v1.33
```

That gets you out of this single rule while keeping every other Baseline check.

## Legitimate use cases for `.host`

If a probe genuinely needs to reach something other than the pod IP (e.g. a sidecar checking an upstream dependency), switch to an **exec probe**:

```yaml
livenessProbe:
  exec:
    command: ["/bin/sh", "-c", "curl -fs http://upstream.example.com/health || exit 1"]
```

The network origin is now the pod, not the kubelet — which is the whole point of the rule.

## Explicit non-goals

- Removing `.host` from the core API. Core API fields don't get removed.
- Replacing it with a safer alternative on the existing types. That work lives in KEP-4559, which introduces new probe types without the field.

## Where this sits relative to other controls

- **NetworkPolicy** doesn't help — probes originate from kubelet, not the pod, so NetworkPolicy never sees them.
- **IMDSv2-style header requirements** on Azure would help against simple SSRF but don't cover the full blast radius (node loopback, control-plane addresses, arbitrary external hosts).
- **Egress firewall on the node** is defense in depth but is rarely fine-grained enough to distinguish kubelet-initiated probes from legitimate kubelet traffic.

The right layer for this is admission, and the right admission is PSA — which is why KEP-4940 put it there instead of shipping yet another Kyverno policy that every operator has to redeploy.
