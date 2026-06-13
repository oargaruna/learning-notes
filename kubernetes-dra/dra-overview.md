# DRA Overview

DRA is a framework for requesting, configuring, and sharing specialized hardware (primarily GPUs,
but also NICs, FPGAs) in a far more expressive way than the old **device plugin** model. It went
**GA in Kubernetes 1.34 (Aug 2025)** after a long alpha/beta evolution.

## Why it exists — the problem with the old model

The legacy approach was **Device Plugins + Extended Resources**:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1   # opaque countable integer
```

Limitations:

- A GPU is just an **opaque counter**. You can't say "give me a GPU with ≥40GB memory" or
  "an H100, not a T4."
- **No structured sharing.** You get whole devices. Time-slicing/MPS/MIG had to be bolted on
  out-of-band via config files baked into the node image.
- **No parameters.** You can't pass configuration (e.g. a MIG profile, a sharing strategy)
  through the request.
- The scheduler is blind to device topology and attributes.

## The DRA object model

DRA borrows the *storage* mental model (PV / PVC / StorageClass) and applies it to devices:

| Storage analogy   | DRA object              | Role                                                                 |
| ----------------- | ----------------------- | -------------------------------------------------------------------- |
| StorageClass      | **DeviceClass**         | A category of devices + a CEL selector (e.g. "all NVIDIA GPUs")      |
| PVC               | **ResourceClaim**       | A request for one or more devices, with constraints/config           |
| (auto-provision)  | **ResourceClaimTemplate** | Stamps out a per-pod claim                                         |
| PV inventory      | **ResourceSlice**       | Published by the driver/kubelet per node: devices + their attributes |

A pod references a claim, and the scheduler matches the claim's requirements against
ResourceSlices using **CEL expressions**:

```yaml
# DeviceClass selects the vendor's devices
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: gpu.nvidia.com
spec:
  selectors:
  - cel:
      expression: device.driver == "gpu.nvidia.com"
---
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: single-gpu
spec:
  spec:
    devices:
      requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.nvidia.com
          selectors:
          - cel:
              expression: device.attributes["gpu.nvidia.com"].productName == "H100"
```

## The key architectural shift: "structured parameters"

The original (1.26) DRA design required each **vendor to run a control-plane controller** that
did allocation — the scheduler couldn't reason about devices itself. That was the blocker to GA.

The redesign moved to **structured parameters**: drivers *publish* device inventory and attributes
into `ResourceSlice` objects, and the **scheduler itself does the allocation** using those
structured descriptions. No vendor scheduler plugin in the critical path → predictable,
debuggable, and GA-able.

## Notable DRA capabilities (1.34+)

- **Attribute/CEL-based selection** — pick devices by model, memory, capabilities.
- **Device sharing** — multiple pods/containers can consume the same claim; **consumable
  capacity** lets a device be split by a quantity.
- **Partitionable devices** — a physical device advertises multiple possible partitions
  (the foundation for dynamic MIG).
- **Prioritized list** — "give me an H100, but fall back to an A100."
- **Admin access** — privileged/monitoring access to devices.
- **ResourceQuota** integration for devices.
