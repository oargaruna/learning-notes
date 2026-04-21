# Azure Identity and Key Protection

Notes on how Azure secures workload identity and private key material across VMs and AKS, with a focus on the threat model of intermediaries (CSI driver, kubelet, host) between Azure Key Vault and the workload.

## Contents

- [msi-on-vms-and-aks.md](./msi-on-vms-and-aks.md) — how Managed Identities work on VMs (IMDS + host-issued fabric cert) and on AKS (Workload Identity via OIDC federation).
- [akv-keys-in-transit.md](./akv-keys-in-transit.md) — what the Secrets Store CSI path actually protects vs. leaves exposed, and the two AKV features that address it.
- [secure-key-release.md](./secure-key-release.md) — deep dive on SKR: release policy, attestation binding via `report_data`, and the end-to-end flow to a TEE.
- [tees-on-aks.md](./tees-on-aks.md) — TEE options available on AKS (SGX, SEV-SNP, TDX, Confidential Containers, Confidential GPU) and when to pick each.
- [non-tee-alternatives.md](./non-tee-alternatives.md) — pragmatic alternatives when confidential computing isn't on the table: non-exportable HSM keys, workload identity, blast-radius reduction, detection.
