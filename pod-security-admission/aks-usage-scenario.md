# Rolling out PSA on a multi-tenant AKS cluster

**Scenario.** Platform team runs a shared AKS cluster. Tenant namespaces are labeled `tenant=*`. Goal: enforce `baseline:v1.34` on every tenant namespace — picks up KEP-4940's probe-host block as a side effect. Must know what breaks before flipping the switch; must be able to roll a single namespace back if something sneaks through.

AKS-specific pieces throughout: audit log goes to Log Analytics, not a file on disk, and `AdmissionConfiguration` isn't user-editable on the managed control plane.

## Step 1 — wire up control-plane audit logs

PSA's `audit` mode writes to the kube-apiserver audit stream. If that stream isn't sunk somewhere, audit mode is silently doing nothing. On AKS, enable it via a diagnostic setting on the cluster resource:

```bash
WS_ID=$(az monitor log-analytics workspace show -n myWs -g myRg --query id -o tsv)
CLUSTER_ID=$(az aks show -n myAks -g myRg --query id -o tsv)

az monitor diagnostic-settings create \
  --name aks-audit \
  --resource "$CLUSTER_ID" \
  --workspace "$WS_ID" \
  --logs '[{"category":"kube-audit-admin","enabled":true}]'
```

Use `kube-audit-admin` during rollout. It excludes high-volume get/list/watch traffic from system components and is cheap enough to leave on. Upgrade to full `kube-audit` only for incident forensics.

## Step 2 — label namespaces in audit + warn mode

```bash
for ns in $(kubectl get ns -l tenant -o name); do
  kubectl label "$ns" --overwrite \
    pod-security.kubernetes.io/audit=baseline \
    pod-security.kubernetes.io/audit-version=v1.34 \
    pod-security.kubernetes.io/warn=baseline \
    pod-security.kubernetes.io/warn-version=v1.34
done
```

`enforce` is deliberately absent. Nothing gets rejected — but:

- Every would-be violation gets an audit annotation.
- `warn` mode: any `kubectl apply` from a tenant returns an inline warning, so the tenant sees the future breakage on their own terminal before the platform team has to email them.
- `audit`/`warn` also cover Deployments/Jobs/CronJobs — you catch the offending workload at the controller, not only at the pod.

Run this for at least a full business cycle (usually a week — covers nightly jobs, weekly cron, deploy cadence).

## Step 3 — query violations in Log Analytics

PSA annotations land on the apiserver audit events:

```kusto
AKSAuditAdmin
| where TimeGenerated > ago(7d)
| extend ann = parse_json(tostring(AuditAnnotations))
| where isnotempty(ann["pod-security.kubernetes.io/audit-violations"])
| project TimeGenerated,
          ns        = tostring(ObjectRef.namespace),
          name      = tostring(ObjectRef.name),
          kind      = tostring(ObjectRef.resource),
          user      = tostring(User.username),
          violation = tostring(ann["pod-security.kubernetes.io/audit-violations"])
| summarize count(), any(name), any(user) by ns, violation
| order by count_ desc
```

Typical findings:

- `probeHost (container ...)` — KEP-4940 hits. Probe has `.host` set.
- `hostPath volumes` — workload mounts a path from the node filesystem.
- `hostNetwork` / `hostPID` / `hostIPC` — misconfigured DaemonSets.
- `allowPrivilegeEscalation != false`, missing `runAsNonRoot` — only if auditing against `restricted`.

Also watch the metric surface (scraped from kube-apiserver by managed Prometheus / Container Insights):

```
apiserver_pod_security_evaluations_total{decision="deny", mode="audit", level="baseline"}
```

## Step 4 — fix or exempt

Per violation, pick one:

**Fix the workload.** Most probe-host hits rewrite cleanly to an exec probe or just drop `.host` and rely on the pod-IP default.

```yaml
# before — blocked at v1.34 baseline
livenessProbe:
  httpGet:
    host: upstream.example.com
    path: /health
    port: 8080

# after — exec probe runs inside the container; kubelet is no longer the originator
livenessProbe:
  exec:
    command: ["/bin/sh", "-c", "curl -fs http://upstream.example.com:8080/health || exit 1"]
```

**Exempt the namespace.** PSA has a formal exemption mechanism via `AdmissionConfiguration` on the apiserver, **but AKS doesn't expose apiserver flags**. Practical workarounds:

- Leave the namespace unlabeled (no enforce label → no enforcement). Suitable for add-on namespaces like `kube-system`, `gatekeeper-system`, CSI-driver namespaces.
- Label the namespace `enforce=privileged` explicitly. Still subject to audit/warn if you want visibility.
- Use a dynamic admission policy (VAP or Kyverno) that runs **after** PSA to grant narrower exceptions if one-off exemptions are needed per-workload.

Don't label `kube-system` with `enforce=baseline` — many managed AKS add-ons (Azure CNI, AKV CSI, GPU operator, monitoring agent) legitimately need `hostPath`, `hostNetwork`, or privileged containers.

## Step 5 — flip to enforce

After the audit log is clean for at least a business cycle:

```bash
for ns in $(kubectl get ns -l tenant -o name); do
  kubectl label "$ns" --overwrite \
    pod-security.kubernetes.io/enforce=baseline \
    pod-security.kubernetes.io/enforce-version=v1.34
done
```

New pods that violate Baseline — including any probe with `.host` set — get rejected at admission. Existing pods keep running until they're recreated (deploy rollout, node rotation, scale event).

## Step 6 — watch for fallout

```
apiserver_pod_security_evaluations_total{decision="deny", mode="enforce", level="baseline"}
```

A spike after flipping enforce usually means:

- A nightly CronJob you didn't exercise during the audit window just fired.
- A Deployment rolled and the fresh pod hit a rule the old pods bypassed.
- A tenant is retrying a rejected apply in a loop.

Pair the metric with a ReplicaFailure watcher — rejected pods surface as:

```
type: ReplicaFailure
reason: FailedCreate
message: 'pods ... is forbidden: violates PodSecurity "baseline:v1.34": probeHost ...'
```

## Rollback — per namespace

If one tenant's workload needs immediate relief:

```bash
kubectl label ns tenant-acme --overwrite \
  pod-security.kubernetes.io/enforce-version=v1.33
```

Keeps every other Baseline rule in place; gets you out of KEP-4940's probe-host check while the tenant fixes the workload. Remove the override when they've switched to an exec probe.

## Quick cheat sheet

```bash
# see a namespace's PSA configuration
kubectl get ns tenant-acme -o jsonpath='{.metadata.labels}' | jq

# dry-run a manifest against a PSA level
kubectl label ns scratch pod-security.kubernetes.io/warn=restricted
kubectl apply -n scratch --dry-run=server -f pod.yaml

# check which AKS-audit category is enabled
az monitor diagnostic-settings list --resource "$CLUSTER_ID" \
  --query "value[].logs[?enabled].category" -o tsv
```

## Gotchas specific to AKS

- **`AdmissionConfiguration` is not user-editable.** If you need the exemption-by-username feature, the answer on AKS is either namespace-scoped labeling or a dynamic admission policy — not the in-tree file.
- **Audit cost.** Log Analytics ingestion is billed by volume. PSA audit annotations themselves are light, but enabling `kube-audit` (not `kube-audit-admin`) at scale on a busy cluster is expensive.
- **Managed add-ons land in `kube-system`**. Treat it as a privileged namespace. Don't assume "baseline everywhere" works — it won't.
- **Version pinning per namespace** is the clean rollback lever. Resist the urge to cluster-wide downgrade or to disable the admission plugin on the cluster (the latter isn't exposed on AKS anyway).
- **Workload Identity doesn't interact with PSA.** PSA operates on pod spec fields; Workload Identity operates on projected SA tokens and federated credentials. The two are orthogonal — enforcing Baseline doesn't affect a pod's ability to get Azure tokens.
