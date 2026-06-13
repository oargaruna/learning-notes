# Kubernetes Dynamic Resource Allocation (DRA)

Notes on DRA — the Kubernetes framework (SIG-Node / WG-Device-Management) for requesting,
configuring, and sharing specialized hardware (GPUs, NICs, FPGAs) far more expressively than
the legacy device-plugin model. DRA core went **GA in Kubernetes 1.34 (Aug 2025)**.

## Contents

- [dra-overview.md](./dra-overview.md) — why DRA exists (limits of device plugins), the object
  model (DeviceClass / ResourceClaim / ResourceClaimTemplate / ResourceSlice), the "structured
  parameters" shift that unblocked GA, and the key 1.34 capabilities.
- [computedomain.md](./computedomain.md) — deep dive on the NVIDIA DRA Driver for GPUs and
  **ComputeDomains**: IMEX, NVLink fabrics (GB200 NVL72), concepts, lifecycle, and practical usage.
