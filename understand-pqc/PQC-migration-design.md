# PQC Migration Design: .NET Kestrel on Kubernetes with Istio

**Date:** 2026-03-23
**Status:** Approved
**Approach:** Hybrid KEM First, Signatures Later (Approach A)

---

## 1. Environment Summary

| Layer | Details |
|---|---|
| **Application** | .NET 9, Kestrel server |
| **Infrastructure** | Kubernetes with Istio (Envoy sidecar) |
| **TLS boundary 1** | Istio ingress gateway — serves internal + external clients |
| **TLS boundary 2** | Istio mTLS mesh — pod-to-pod communication |
| **Certificates** | RSA-based; internal CA for mTLS, public CA for external |
| **Cert storage** | Azure Key Vault via Secrets Store CSI Driver |
| **Data sensitivity** | General business data with multi-year confidentiality requirements |
| **Migration driver** | Harvest-now-decrypt-later concern + proactive readiness |

---

## 2. Migration Architecture

The migration targets the **key exchange** algorithm (KEM) at both TLS boundaries, moving from classical-only to hybrid (classical + PQC). Certificates and signatures remain RSA in Phase 1.

```
External Clients
       │
       ▼
┌─────────────────────────┐
│  Istio Ingress Gateway  │  ← Layer 1: Hybrid KEM for external TLS
│  (Envoy + BoringSSL)    │
└──────────┬──────────────┘
           │
       ┌───▼───┐
       │ Istio │
       │ mTLS  │           ← Layer 2: Hybrid KEM for mesh-internal TLS
       │ mesh  │
       └───┬───┘
           │
┌──────────▼──────────────┐
│  Kestrel (.NET 9)       │  ← Layer 3: No TLS changes needed (Istio terminates)
│  App container           │
└──────────┬──────────────┘
           │
┌──────────▼──────────────┐
│  AKV + CSI Driver       │  ← Layer 4: No changes in Phase 1 (certs stay RSA)
│  Certificate storage     │
└─────────────────────────┘
```

**Key insight:** Because Istio owns both TLS boundaries, this is primarily an **infrastructure migration**, not an application code change. Kestrel and the AKV/CSI pipeline are untouched in Phase 1.

---

## 3. Layer 1 — Istio Ingress Gateway (External TLS)

### Why This Is Highest Priority

This layer is exposed to external traffic that could be harvested by adversaries. The harvest-now-decrypt-later threat applies directly here.

### What Changes

Envoy (powering Istio's ingress gateway) uses **BoringSSL** for TLS. BoringSSL includes ML-KEM support. Required upgrades:

1. **Istio version:** Upgrade to a version whose bundled Envoy supports PQC hybrid key exchange (Istio 1.20+ with PQC-capable BoringSSL).
2. **Ingress TLS configuration:** Add hybrid key exchange curves (X25519Kyber768) to the ingress gateway via `EnvoyFilter` or Gateway resource.
3. **No certificate changes:** The existing RSA certificate from the public CA continues to work. Key exchange algorithm is independent of the certificate's signature algorithm.

### How the Hybrid Handshake Works

```
Client                          Ingress Gateway
  │                                    │
  │── ClientHello ──────────────────►  │
  │   (supported_groups:               │
  │    X25519Kyber768, X25519, ...)    │
  │                                    │
  │  ◄─── ServerHello ────────────── │
  │       (selected: X25519Kyber768)   │
  │                                    │
  │   [Hybrid KEM exchange happens]    │
  │   shared_secret = KDF(             │
  │     X25519_secret || ML-KEM_secret │
  │   )                                │
  │                                    │
  │   [Server signs with RSA cert]     │
  │   (signature algorithm unchanged)  │
```

### Backward Compatibility

This is safe for clients that don't support PQC:

- Client offers `X25519Kyber768` → hybrid KEM is negotiated
- Client only supports `X25519` or `P-256` → classical key exchange is negotiated
- No client breaks. PQC is additive, not replacing.

---

## 4. Layer 2 — Istio mTLS Mesh (Pod-to-Pod)

### What Changes

1. **Mesh-wide TLS config:** Configure Istio's `MeshConfig` or `DestinationRule` to prefer hybrid key exchange for sidecar-to-sidecar mTLS.
2. **Same Istio/Envoy version:** The sidecars use the same Envoy binary as the ingress gateway, so the version upgrade from Layer 1 covers this layer too.
3. **No certificate changes:** Istio's internal SPIFFE certs remain as-is (typically ECDSA). The short-lived nature of these certs (often 24h TTL) already limits their exposure.

### Risk Profile vs. Layer 1

| Factor | External (Layer 1) | Mesh Internal (Layer 2) |
|---|---|---|
| **Who can harvest?** | Anyone on the network path | Only someone inside the cluster network |
| **Urgency** | High — internet-exposed | Lower — requires cluster compromise |
| **Backward compat concern** | External clients you don't control | All sidecars — you control both sides |
| **Rollout** | Ingress gateway only (single point) | Every sidecar in the mesh |

### Rollout Strategy

Because you control both sides of every mesh connection, you can roll out more aggressively than Layer 1. However, the blast radius is larger (every pod).

1. **Namespace-by-namespace:** Use `DestinationRule` scoped to individual namespaces to enable hybrid KEM incrementally
2. **Canary first:** Pick a low-risk namespace, enable hybrid KEM, monitor for TLS handshake failures or latency regressions
3. **Mesh-wide:** Once validated, apply globally via `MeshConfig`

### Performance Consideration

Hybrid KEM adds ~1KB to the TLS handshake and a small CPU cost for ML-KEM encapsulation/decapsulation (sub-millisecond on modern hardware). For most services this is negligible, but for very high-throughput, latency-sensitive services with frequent new connections, monitor:

- Handshake latency (p99)
- Envoy sidecar CPU usage
- Connection setup rate

---

## 5. What Doesn't Change (Layers 3 & 4)

### Layer 3: Kestrel Application

**No changes required in Phase 1.**

Kestrel receives plaintext traffic from the Istio sidecar (or mTLS-terminated traffic). It never negotiates TLS directly with external clients or peer services — Envoy handles that. Application code, Kestrel TLS configuration, and `appsettings.json` stay untouched.

**When this changes:** In Phase 2 (signature migration), if switching to ML-DSA certificates, you'll need to verify:
- .NET 9's `SslStream` / Kestrel can load and use ML-DSA certs (depends on OpenSSL 3.x support in the container base image)
- The Secrets Store CSI Driver can mount the new cert formats correctly

### Layer 4: AKV + Secrets Store CSI Driver

**No changes required in Phase 1.**

RSA certificates remain valid for signing/authentication. The CSI driver continues to fetch them from AKV and mount them into pods exactly as it does today.

**When this changes:** Phase 2 (signature migration) will require:
- Internal CA to issue hybrid or ML-DSA certificates
- AKV to support storing new cert formats (PFX/PEM with PQC key types)
- CSI driver to handle any new key material sizes

### Phase 1 Change Footprint

| Component | Changes? | What |
|---|---|---|
| Istio version | **Yes** | Upgrade to PQC-capable Envoy/BoringSSL |
| Ingress Gateway config | **Yes** | Add hybrid KEM curves |
| Mesh mTLS config | **Yes** | Enable hybrid KEM for sidecar-to-sidecar |
| Kestrel / .NET app | No | — |
| AKV certificates | No | — |
| CSI Driver | No | — |
| Internal CA | No | — |
| Public CA | No | — |

---

## 6. Rollout Plan & Validation

### Phased Rollout

```
Phase 1a: Istio Upgrade
    │
    ▼
Phase 1b: Ingress Gateway — Hybrid KEM (external TLS)
    │
    ▼
Phase 1c: Mesh mTLS — Hybrid KEM (canary namespace)
    │
    ▼
Phase 1d: Mesh mTLS — Hybrid KEM (all namespaces)
    │
    ▼
Phase 2 (future): Signature migration (RSA → ML-DSA)
```

### Phase 1a: Istio Upgrade

- Upgrade Istio control plane and data plane (sidecars) to a version with PQC-capable Envoy
- Use canary revision strategy (`istioctl install --revision`)
- **Validation:** Confirm Envoy binary supports hybrid KEM by checking `envoy --version` or BoringSSL build flags in the proxy container

### Phase 1b: Ingress Gateway — Hybrid KEM

- Apply `EnvoyFilter` or Gateway config adding `X25519Kyber768` to the ingress gateway's TLS curves
- **Validation:**
  - Use `openssl s_client` (OpenSSL 3.5+ or PQC-patched build) to connect and verify hybrid KEM is negotiated
  - Confirm classical-only clients still connect successfully (fallback works)
  - Monitor ingress gateway error rates and handshake latency for 48-72h before proceeding

### Phase 1c: Mesh mTLS — Canary Namespace

- Pick a low-traffic, non-critical namespace
- Apply `DestinationRule` with hybrid KEM for that namespace
- **Validation:**
  - Check Envoy access logs / stats for TLS handshake success rate
  - Compare p99 latency before/after
  - Monitor sidecar CPU for unexpected spikes
  - Run for 1 week before expanding

### Phase 1d: Mesh mTLS — All Namespaces

- Apply hybrid KEM mesh-wide via `MeshConfig`
- **Validation:**
  - Same metrics as 1c across the full mesh
  - Alert on any handshake failure rate increase

### Rollback

At every phase, rollback is straightforward:
- **1b:** Remove the `EnvoyFilter` / revert Gateway config — ingress falls back to classical KEM
- **1c/1d:** Remove the `DestinationRule` or `MeshConfig` change — mesh reverts to classical mTLS
- No certificate re-issuance, no data migration, no application restarts needed

### Monitoring Checklist

| Metric | Where | Alert Threshold |
|---|---|---|
| TLS handshake failure rate | Envoy stats (`ssl.handshake`) | Any increase over baseline |
| Handshake latency (p99) | Envoy stats / Istio telemetry | >10ms regression |
| Sidecar CPU | Kubernetes metrics | >15% increase |
| Connection setup rate | Envoy stats | Drop from baseline |

---

## 7. Future Work (Phase 2: Signature Migration)

Phase 2 migrates certificates from RSA to hybrid or ML-DSA signatures. This is **not urgent** (no harvest-now threat for signatures) and is **blocked** on ecosystem readiness:

- Public CAs issuing ML-DSA or hybrid certificates
- AKV supporting PQC key types
- .NET / OpenSSL supporting ML-DSA certificate loading
- Istio / Envoy supporting ML-DSA for mTLS cert validation

This phase will require its own design document when the ecosystem is ready.
