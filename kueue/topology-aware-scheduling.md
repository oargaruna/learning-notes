# Kueue & Topology-Aware Scheduling (TAS)

## What Kueue is

Kueue is a Kubernetes-native **job queueing** system (Kubernetes SIG-Scheduling). It does
**admission control and quota management**, *not* pod placement.

- Kueue answers: *"should this whole job be allowed to start now, given quotas and fairness?"*
- kube-scheduler still answers: *"which node does each pod go on?"*

Mechanism: the Job `suspend` field. You submit a Job (or JobSet, MPIJob, RayJob, plain Pod,
etc.) suspended. Kueue holds it in a queue until quota is free, then **unsuspends** it and
normal scheduling proceeds. Gating happens at the *workload* (all-or-nothing) level — exactly
what distributed ML/HPC jobs need, since a half-scheduled training job is useless.

### Core objects

| Object | Scope | Role |
|---|---|---|
| **ResourceFlavor** | cluster | A *variant* of resource (e.g. `nvidia-a100`, `spot`, `arm`); binds to nodes via labels/taints |
| **ClusterQueue** | cluster | Quota pool: how much CPU/mem/GPU per flavor, plus preemption & borrowing rules |
| **LocalQueue** | namespace | What users submit to; points at a ClusterQueue |
| **Workload** | namespace | Kueue's internal wrapper around your Job; the thing it tracks/admits |
| **Cohort** | cluster | Group of ClusterQueues that lend/borrow unused quota between each other |

Flow: submit Job → Kueue creates a Workload → Workload waits in a LocalQueue → ClusterQueue
admits it when quota is free (borrowing from cohort or preempting lower-priority work if
needed) → Job unsuspended.

## Topology-Aware Scheduling (TAS)

TAS lets Kueue care about *where* pods land relative to physical network topology, decided
**at admission time**.

### Why it exists

For distributed ML/HPC, network distance between pods dominates performance. Same node
(NVLink), same rack (top-of-rack switch), and same datacenter block have wildly different
bandwidth/latency. Plain kube-scheduler has no notion of "keep these 64 pods within one
rack." TAS adds it.

### Modeling topology

1. **Label nodes** with a hierarchy, e.g.:
   - `cloud.provider.com/topology-block`
   - `cloud.provider.com/topology-rack`
   - `kubernetes.io/hostname`

2. **Define a `Topology`** listing levels widest → narrowest:

   ```yaml
   apiVersion: kueue.x-k8s.io/v1alpha1
   kind: Topology
   metadata:
     name: gpu-topology
   spec:
     levels:
     - nodeLabel: cloud.provider.com/topology-block
     - nodeLabel: cloud.provider.com/topology-rack
     - nodeLabel: kubernetes.io/hostname
   ```

3. **Point a ResourceFlavor at it** via `.spec.topologyName: gpu-topology`. Any quota drawn
   from that flavor is now topology-aware.

### Requesting it on a workload

Annotate the PodSet (on the Job's pod template):

- `kueue.x-k8s.io/podset-required-topology: "<level>"` — **hard** constraint. All pods must
  fit within a single domain at that level (e.g. one rack), or the workload isn't admitted.
- `kueue.x-k8s.io/podset-preferred-topology: "<level>"` — **best effort**. Try to pack within
  that level; if it doesn't fit, walk **up** the hierarchy (rack → block → …) to the
  narrowest level that does.
- `kueue.x-k8s.io/podset-unconstrained-topology: "true"` — no locality requirement (TAS still
  tracks/accounts capacity).

Newer versions add **slice**-based annotations (`podset-slice-required-topology` +
`podset-slice-size`) to require locality per group of N pods rather than the whole PodSet.

### Under the hood at admission

1. TAS keeps a **snapshot of free capacity per topology domain** — each node's allocatable
   resources and current usage, rolled up the tree (host → rack → block).
2. On admission it runs a **best-fit search over the topology tree**: for `required` it checks
   the PodSet fits inside a single domain at that level; for `preferred` it starts at the
   requested level and climbs until it fits. It prefers the tightest-fitting domain to avoid
   fragmenting large contiguous regions.
3. It assigns specific pods to specific domains and records that in the Workload status.
4. Pods are created with a **scheduling gate**. Kueue's **TopologyUngater** controller injects
   the right `nodeSelector`/affinity for the assigned domain and removes the gate, forcing
   kube-scheduler to bind each pod onto nodes within its assigned domain.

**Division of labor:** Kueue picks the topology domain (and reserves capacity);
kube-scheduler does final node binding within that domain. For indexed/ranked workloads
(e.g. JobSet) TAS can align placement with pod indexes so ranks land predictably.

### Caveats

- TAS is an alpha/evolving feature behind a feature flag, maturing across the 0.9 → 0.13
  line — check your version for exact annotation names and the `Topology` API
  group/version (`v1alpha1` at time of writing).
- Capacity tracking is Kueue's own view; node changes it doesn't see (DaemonSets, system
  reserved) can cause occasional admit-then-fail-to-place mismatches.
- TAS only constrains placement within a **single ResourceFlavor's** topology.
