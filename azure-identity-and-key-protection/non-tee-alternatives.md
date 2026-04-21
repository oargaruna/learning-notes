# Replicating Key Protection Without TEE

## What you lose without TEE

TEE-backed SKR gives two properties nothing else reproduces:

1. **Host cannot read workload memory.** Node admin, kubelet, and host kernel are all out of the TCB.
2. **Keys bound to a hardware-measured workload identity.** Swap the image, release fails.

Without TEEs, a sufficiently privileged attacker on the node always wins eventually. The game is no longer "cut intermediaries out" — it's "make the key worthless to them, or make them never see it."

## Biggest win: don't let the key leave the HSM

The single most underused alternative. No TEE needed.

- AKV Premium or Managed HSM, key created with `exportable=false`.
- Workload authenticates via workload identity (federated OIDC, no stored secret) and calls `/sign`, `/encrypt`, `/wrapKey`, `/unwrapKey` over TLS.
- CSI driver, kubelet, node — none ever see the private key. Only the outputs of operations using the key.

### Fits well for

- JWT / OIDC / SAML signing
- mTLS handshake offload via a localhost sidecar that terminates TLS by calling AKV for the ECDHE signature
- Envelope encryption: DEK per object, wrap/unwrap via AKV, bulk crypto local
- Code signing, certificate issuance

### Doesn't fit

- Bulk encryption at line rate (latency + throttling)
- Anywhere the plaintext key is the deliverable (SSH host keys, WireGuard, disk encryption root keys)

Managed HSM also gives FIPS 140-3 L3, BYOK via nCipher/Thales, and per-key RBAC with "secure key release" permissions that can be audited independently.

## Second biggest win: architect the secret away

Every secret not stored is a secret that can't leak.

- **Workload Identity (OIDC federation)** for Azure, AWS, GCP, GitHub, Vault — no client secrets at all.
- **SPIFFE / SPIRE on AKS** for service-to-service auth — short-lived SVIDs (minutes) issued to attested workloads. Node agent attests pods by kubelet selectors (image digest, SA, namespace). Not hardware attestation, but cryptographic workload identity without stored secrets.
- **cert-manager + AKV issuer** for TLS certs — short TTL (hours), auto-rotated, private keys generated in-cluster per pod rather than distributed.
- **Database auth via Entra ID tokens** instead of passwords (Postgres, SQL, Cosmos all support this).

Goal: AKV ends up holding only the handful of keys that genuinely must exist (root signing keys, BYOK roots, long-lived partner API keys). Everything else is derived just-in-time from workload identity.

## Shrink blast radius at runtime

If a key must land on the node, these reduce but don't eliminate exposure:

| Control | What it gets you |
|---|---|
| **Dedicated node pool** + node taints for sensitive workloads | No noisy-neighbor DaemonSets reading tmpfs |
| **Secrets Store CSI with `syncSecret: false`** | Material stays in pod tmpfs, never reaches etcd |
| **KMS plugin for etcd encryption** (AKV as KMS) | Encrypts secrets at rest in etcd; closes backup/snapshot leaks |
| **Pod Security Standards: restricted** | No hostPath, no host namespaces, no privileged pods that could `nsenter` others |
| **AppArmor / seccomp / SELinux profiles** | Narrows what a compromised pod can syscall |
| **Kata Containers (without SEV-SNP)** | Each pod in its own lightweight VM → kernel isolation between pods. Protects against pod-to-pod breakout, not against host admin |
| **gVisor** | User-space kernel shim; similar pod-isolation story to Kata, cheaper, weaker |
| **Read-only root FS + no shell in image** (distroless) | Harder to stage an exfil after compromise |
| **Sidecar secret broker** (Vault Agent, akv2k8s) | Secret lives in one process's memory, not on disk; rotate without restart |

**Kata-without-SEV** is worth calling out specifically. Gives you the pod-boundary hypervisor that CoCo uses, minus the memory encryption. Stops a compromised pod A from reading pod B's secrets, even on the same node. Good intermediate step before committing to SEV-SNP hardware.

## Lifecycle controls — make stolen keys expire fast

- **AKV auto-rotation** + short TTL on the consuming side.
- **Secrets Store CSI `--enable-secret-rotation`** with a 1–5 minute poll.
- **Just-in-time access**: Azure PIM / Entra Privileged Access Groups so even AKV access policies are time-bounded for humans. Workloads use managed identity continuously; humans don't.
- **Event Grid on AKV** → alert on `SecretNewVersionCreated`, `SecretNearExpiry`, unusual `Get` patterns.
- **AKV firewall + private endpoint**: only the AKS subnet can reach AKV. Stolen credentials useless from outside.

## Detection as compensating control

Since you can't prevent a privileged node attacker, invest in noticing:

- **AKV diagnostic logs → Sentinel / Log Analytics**: every `Get`, `Sign`, `Release`. Alert on new caller identities, unusual volumes, off-hours access.
- **Falco / Tetragon** on nodes watching for reads of CSI tmpfs paths by non-expected PIDs.
- **Audit `kubectl exec` and `kubectl cp`** into sensitive namespaces.
- **Canary secrets** — a plausible-looking unused AKV secret whose `Get` event is a known-bad signal.

## A realistic stack without TEE

For most teams, this gets ~80% of the way:

1. **Workload Identity everywhere** → no stored secrets for platform auth.
2. **Managed HSM with non-exportable keys** for the few real secrets → HSM does the crypto.
3. **SPIFFE/SPIRE or cert-manager** for service identity → short-lived certs.
4. **Dedicated node pool** for anything touching unavoidable plaintext secrets, locked down with PSS restricted + AppArmor.
5. **CSI with rotation + `syncSecret: false`** + AKV private endpoint.
6. **Kata (no SEV)** if pod-to-pod kernel isolation matters on shared nodes.
7. **AKV + node audit logs wired to detection**.

You still lose to a determined host-level attacker. But you've eliminated the "oops a secret sat in a Kubernetes Secret forever" failure mode, which is where almost all real-world key compromises actually come from.
