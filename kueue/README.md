# Kueue

Notes on Kueue — the Kubernetes-native job queueing system (SIG-Scheduling) that does
admission control and quota management for batch/ML/HPC workloads.

## Contents

- [topology-aware-scheduling.md](./topology-aware-scheduling.md) — what Kueue is (suspend-based
  admission, ResourceFlavor/ClusterQueue/LocalQueue/Workload/Cohort), and how Topology-Aware
  Scheduling packs a job's pods within a physical topology domain (rack/block) at admission time.
