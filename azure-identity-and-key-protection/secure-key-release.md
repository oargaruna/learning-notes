# Secure Key Release (SKR)

## The core idea

You want a private key inside a workload, but you don't trust the host, kubelet, CSI driver, or cluster admin. SKR flips the usual model:

- **Usual model**: "authenticate, then get the key" — identity-based.
- **SKR**: "prove what code is running in what hardware state, then get the key" — attestation-based.

The key is cryptographically bound to a measurement of the code/environment. If the measurement changes, release fails.

## Components

| Component | Role |
|---|---|
| **TEE** (AMD SEV-SNP VM, Intel TDX VM, or SGX enclave) | Executes the workload in an encrypted memory region the host can't read |
| **Hardware root of trust** | CPU-signed attestation report containing measurements (firmware hash, VM launch digest, enclave MRENCLAVE) |
| **Microsoft Azure Attestation (MAA)** | Validates the raw HW report against AMD/Intel roots, re-issues a JWT with normalized claims, signed by MAA's key |
| **AKV key with `release` permission + release policy** | Key marked `exportable=false` but `releasable=true` under a policy |
| **Release policy (JSON)** | Rules DSL: "release iff JWT's `x-ms-sevsnpvm-*` claims match these values" |

## The release policy

JSON with `anyOf` / `allOf` grammar over JWT claims. SEV-SNP example:

```json
{
  "version": "1.0.0",
  "anyOf": [{
    "authority": "https://sharedeus.eus.attest.azure.net",
    "allOf": [
      { "claim": "x-ms-attestation-type", "equals": "sevsnpvm" },
      { "claim": "x-ms-compliance-status", "equals": "azure-compliant-cvm" },
      { "claim": "x-ms-sevsnpvm-hostdata", "equals": "aaaa...bbbb" },
      { "claim": "x-ms-sevsnpvm-is-debuggable", "equals": "false" }
    ]
  }]
}
```

Key fields:

- **`authority`** — which MAA instance AKV will accept JWTs from.
- **`x-ms-sevsnpvm-hostdata`** — 32 bytes of user data pushed into the launch measurement. For Confidential Containers, this is how Kata binds the pod spec / container image digest into the HW report. **This is the anchor** that says "only this workload gets the key."
- **`is-debuggable: false`** — refuses to release to a debug-mode enclave.
- Version / SVN claims let you forbid release to vulnerable firmware.

## End-to-end flow

```
pod (inside SEV-SNP CVM)                host/AKS           MAA              AKV
----------------------------            ----------         ---              ---
1. generate ephemeral RSA keypair
   in TEE memory (pub_e, priv_e)

2. ask CPU: "attest me, and
   include SHA256(pub_e) in
   report_data"
          │
          ├── HW report (AMD-signed, contains
          │   measurements + report_data=hash(pub_e))
          ▼
3. POST report + pub_e to MAA ─────────────────►
                                                 validates vs AMD roots,
                                                 checks is-debuggable, etc.
                                                 returns JWT with claims
          ◄───────────────────────────────────── (signed by MAA)

4. call AKV /keys/{name}/release
   with the MAA JWT  ────────────────────────────────────────────────►
                                                                       - verify JWT sig against MAA JWKS
                                                                       - eval release policy
                                                                       - extract pub_e from JWT
                                                                         (embedded as `x-ms-runtime.keys`)
                                                                       - wrap target key with pub_e
                                                                         (RSA-OAEP over an AES-KW CEK)
   ◄──────────────────────────────────────────────────────────────── returns wrapped key blob

5. priv_e decrypts wrapped key
   inside TEE. Plaintext key
   never touches host memory.
```

### The crucial binding

Step 2: `report_data = hash(pub_e)`. The CPU signs both the code measurement **and** the ephemeral public key together, so an attacker can't replay someone else's attestation with their own wrapping key.

## Cryptographic detail on step 4

AKV returns a JWE where:

- CEK is an AES-256 key, wrapped to `pub_e` via RSA-OAEP-256.
- Target key material is wrapped to the CEK via AES-KW or AES-GCM.
- JWE header includes `kid` of the AKV key and the MAA-issued JWT in `attestation` field.

Even in flight across the network and sitting in CSI-adjacent memory, the response is encrypted to a public key whose private half exists only inside the enclave.

## Deployment shapes on AKS

**(a) Confidential VM node pool** — whole VM is a SEV-SNP CVM. All pods on the node share the TEE. SKR client is the app (or a sidecar) using Azure SDK's `release_key`. Binding granularity: the node. Weaker — a co-tenant pod on the same node can read memory.

**(b) Confidential Containers (Kata + SEV-SNP)** — each pod gets its own lightweight CVM. The Kata agent inside measures the pod's OCI image layers and container config into `hostdata`. Release policy can pin to a specific container image digest + command line. This is the shape you want for per-workload key isolation.

Typical CoCo setup:

1. Build an image, note its digest.
2. Generate the expected `hostdata` value (Microsoft ships `genpolicy`, which computes it from the pod YAML).
3. Create AKV key with release policy pinning `x-ms-sevsnpvm-hostdata` to that value.
4. Deploy with `runtimeClassName: kata-cc-isolation`.
5. App calls AKV release endpoint at startup; key lives only in encrypted pod memory.

## Limitations and sharp edges

- **Managed HSM recommended for production**; AKV Premium works but the source key sits in a shared HSM partition.
- **MAA is a trust anchor you're pinning to Microsoft**. You can run your own attestation service, but then you own its key management.
- **Rotation is painful**: rebuilding the image changes `hostdata` → policy must update. Plan for policy as a managed artifact (GitOps, signed by release process).
- **Side channels**: SEV-SNP and TDX have published side-channel papers. TEEs raise the bar; they don't make host compromise irrelevant for sufficiently patient attackers.
- **Debugging is hard** — can't attach a debugger to a non-debuggable enclave, and enabling debug flips `is-debuggable=true`, which policy should reject. Keep a separate staging key with a looser policy.
- **Latency**: release happens once at startup (cache plaintext in TEE memory); no per-op round trip like non-exportable HSM keys.

## Net effect

CSI driver, kubelet, and host kernel handle only JWE blobs. Plaintext exists exclusively inside CPU-enforced encrypted memory bound to a measurement you chose.
