# AKV Private Keys in Transit — the CSI Path Problem

## The default path isn't end-to-end

The Secrets Store CSI driver + Azure Key Vault provider give TLS from AKV to the driver, but **not** end-to-end protection:

1. Driver calls AKV over TLS (workload identity for auth).
2. Driver receives plaintext, hands it to kubelet.
3. Kubelet writes it to a **tmpfs volume** in the pod.

CSI driver, kubelet, privileged DaemonSets, and anyone with node-level access (host SSH, node-root compromise) can read it. AKV's API contract ends at the driver.

## Two AKV features that actually fix this

### 1. Non-exportable keys — don't move the key at all

- Create the key in **AKV Premium (HSM-backed)** or **Managed HSM** with `exportable=false`.
- Private key material **never leaves the HSM boundary**. Not to the CSI driver, not to the pod.
- Workload calls AKV crypto API (`/sign`, `/decrypt`, `/unwrapKey`) over TLS using its managed identity.
- For certificates: the public cert is mounted; the private key stays in AKV.

**Trade-off**: every crypto op is a network round trip. Fine for JWT signing, TLS handshake offload via a sidecar, envelope encryption. Not fine for high-throughput bulk crypto.

### 2. Secure Key Release (SKR) — release only into an attested TEE

See [secure-key-release.md](./secure-key-release.md). Suitable when the workload genuinely needs the key in-process.

- Attach a **key release policy** to the AKV key.
- Workload runs in a TEE (Confidential Containers, Confidential VM, SGX enclave).
- TEE presents an attestation token from **Microsoft Azure Attestation (MAA)** plus an ephemeral wrapping public key.
- AKV validates attestation, then **wraps the key to the TEE's public key**. Decrypted only inside the enclave.
- CSI driver, kubelet, host kernel see only ciphertext. This is the only setup where AKS intermediaries are cryptographically cut out of the path.

## Lesser mitigations on the standard CSI path

These do not remove plaintext from node memory, but reduce exposure window / surface area:

- **`syncSecret: false`** on the SecretProviderClass — keeps material in pod tmpfs, out of etcd.
- **Auto-rotation** (`--enable-secret-rotation`) with short poll interval — leaked value has a short useful life.
- **Workload Identity for the driver** — no service principal secret on the node.
- **Dedicated node pools** for sensitive workloads with tight RBAC, no privileged DaemonSets, locked-down host access.
