# NVIDIA DRA Driver & ComputeDomains

The **NVIDIA DRA Driver for GPUs** (`gpu.nvidia.com` / formerly `k8s-dra-driver`, now shipped via
the GPU Operator) is the reference vendor implementation of DRA. It exposes two driver families:

1. **GPU driver (`gpu.nvidia.com`)** — advertises each GPU as a ResourceSlice device with rich
   attributes (product name, memory, compute capability, MIG support), and enables **dynamic MIG**
   (MIG profiles created on demand per ResourceClaim instead of statically at node boot) and
   **declarative GPU sharing** (time-slicing / MPS through the claim, not node-level config files).

2. **ComputeDomain driver (`resource.nvidia.com`)** — the focus of this note. It manages
   **ComputeDomains**, which provision **IMEX** channels for multi-node NVLink workloads.

---

## Background: the hardware problem ComputeDomains solve

### NVLink, NVL72, and Multi-Node NVLink (MNNVL)

On a single DGX/HGX node, ~8 GPUs are wired together with **NVLink** + an on-board NVSwitch, so
they can read/write each other's HBM directly (no PCIe, no network). That's a single-node
**NVLink domain**.

**GB200 NVL72** extends this across nodes: 72 Blackwell GPUs (36 Grace CPUs) in a rack are joined
by an external **NVLink Switch system** into *one* coherent NVLink domain. GPUs on *different
nodes* can now address each other's memory over NVLink. This cross-node capability is **Multi-Node
NVLink (MNNVL)**.

### IMEX — Internode Memory Exchange

For GPUs on different nodes to share memory over the NVLink fabric, the CUDA driver needs a trust +
control mechanism. That's **IMEX (Internode Memory Exchange)**:

- An **IMEX domain** is a set of nodes that are allowed to export/import GPU memory to each other
  over NVLink. Establishing it requires an `nvidia-imex` daemon running on each participating node,
  all configured with the same membership.
- An **IMEX channel** is the access gate *inside* a domain. It surfaces as a device node
  (`/dev/nvidia-caps-imex-channels/channelN`). A process must have a channel injected to actually
  perform cross-node memory operations (e.g. NCCL's MNNVL transport, or CUDA fabric memory handles
  via `cuMemExportToShareableHandle` / `cudaImportExternalMemory`).

**The pre-Kubernetes pain:** IMEX domains were configured *statically* — you'd hand-write the node
membership and start daemons across the whole rack. Two big problems:

- **No isolation.** A static rack-wide IMEX domain means every GPU job could, in principle, reach
  every other job's exported memory. Bad for multi-tenant clusters.
- **Doesn't follow the scheduler.** Kubernetes decides *at admission time* which nodes a job lands
  on. A static, pre-baked IMEX domain can't know that placement in advance.

---

## ComputeDomain: the core concept

A **ComputeDomain** decouples the *physical* NVLink topology (static hardware) from the *logical*
IMEX trust domain (dynamic, per-workload). It is a CRD (`resource.nvidia.com`, e.g.
`kind: ComputeDomain`) that says: "create an isolated IMEX domain, scoped to the pods of *this*
workload, on whatever nodes the scheduler places them, and tear it down when they're gone."

### What the driver does under the hood

1. You create a `ComputeDomain` object and a workload that consumes its **channel**
   ResourceClaim(Template).
2. The scheduler places the workload pods (DRA allocation picks GPUs/channels). Placement is now
   known.
3. The ComputeDomain controller deploys **IMEX daemon pods** (a DaemonSet) onto exactly the nodes
   where the workload landed, and configures them into a single IMEX domain whose membership
   matches that placement.
4. It injects an **IMEX channel device** into each workload pod (via the consumed ResourceClaim).
5. NCCL/CUDA inside the pods detect the channels and use the **MNNVL transport** over NVLink.
6. When the workload completes/scales down, the IMEX domain and its daemons are **torn down**.

So the IMEX domain is *created to match the workload*, not the other way around. That's the whole
trick — and it's why this only works cleanly through DRA, where allocation and placement are
first-class.

### Key vocabulary

| Term            | Meaning                                                                                  |
| --------------- | ---------------------------------------------------------------------------------------- |
| NVLink domain   | Physical set of GPUs wired by NVLink/NVSwitch (static hardware).                          |
| MNNVL           | Multi-Node NVLink — NVLink reaching across nodes (e.g. GB200 NVL72).                      |
| IMEX domain     | Logical set of nodes permitted to share GPU memory over NVLink (dynamic, per-workload).  |
| IMEX channel    | Per-process access gate (`/dev/nvidia-caps-imex-channels/channelN`) inside a domain.     |
| ComputeDomain   | The DRA CRD that provisions/scopes/lifecycles an IMEX domain for one workload.           |

---

## Practical usage

### Illustrative manifests

> Note: the exact `apiVersion` and field names track the driver release (it has moved through
> `v1beta1` etc.) — treat this as the shape, verify against the installed CRD.

```yaml
# 1) Declare a ComputeDomain sized to the job
apiVersion: resource.nvidia.com/v1beta1
kind: ComputeDomain
metadata:
  name: training-cd
spec:
  numNodes: 2                       # how many nodes the IMEX domain should span
  channel:
    resourceClaimTemplate:
      name: training-cd-channel     # the claim template workload pods will reference
```

```yaml
# 2) Workload pods consume the channel claim (here a 2-pod, multi-node job).
#    Each pod also requests GPUs via the normal gpu.nvidia.com DeviceClass.
spec:
  template:
    spec:
      resourceClaims:
      - name: imex-channel
        resourceClaimTemplateName: training-cd-channel
      containers:
      - name: trainer
        image: nvcr.io/.../nccl-tests   # or your training image
        resources:
          claims:
          - name: imex-channel
```

Workloads are typically expressed with something that gangs pods across nodes — an **MPIJob**,
**LeaderWorkerSet**, **JobSet**, or a Kubeflow/PyTorch distributed job — so all ranks land
together and join the same ComputeDomain.

### Where you actually use it

1. **Multi-node LLM training across an NVL72 domain.** Large dense or **MoE** models whose
   parameters/optimizer state are sharded across more GPUs than one node holds. NCCL all-reduce /
   all-to-all runs over NVLink (MNNVL) instead of InfiniBand, dramatically cutting collective
   latency. The ComputeDomain gives that job its own isolated IMEX domain.

2. **Multi-node / disaggregated inference.** Serving very large models (e.g. DeepSeek-R1, Llama
   405B, big MoEs) where a single node's NVLink domain isn't enough, or **prefill/decode
   disaggregation** and **KV-cache sharing** across nodes. Cross-node KV transfer over NVLink needs
   IMEX → a ComputeDomain scopes it to the serving deployment.

3. **Any CUDA app using fabric memory handles across nodes** — e.g. custom kernels that
   `cuMemExportToShareableHandle` and import on a peer node. The ComputeDomain provides the channel
   that authorizes those exports/imports.

### Operational notes / gotchas

- **Hardware prerequisite:** ComputeDomains are only meaningful on MNNVL hardware (GB200/GB300
  NVL72-class systems with the NVLink Switch fabric). On ordinary nodes there's no cross-node
  NVLink to gate.
- **Sizing:** `numNodes` should match the workload's node fan-out; the IMEX domain forms once the
  daemons on all member nodes are up — workload pods may wait on that readiness.
- **Isolation is the headline benefit:** one ComputeDomain per workload = per-tenant blast radius,
  versus a static rack-wide IMEX config.
- **Lifecycle is automatic:** domain + daemons are created on demand and cleaned up on completion;
  don't hand-manage `nvidia-imex`.
- **Install path:** deploy via the **GPU Operator** (which manages the NVIDIA DRA driver), and the
  cluster must have DRA enabled (GA in 1.34; feature-gated/beta in 1.32–1.33).
