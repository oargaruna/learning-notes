# Post-Quantum Cryptography: A Comprehensive Overview

This document covers the landscape of post-quantum cryptography (PQC) — the algorithms designed to resist attacks from both classical and quantum computers. It builds on the foundational context from BB84 and Shor's algorithm, exploring how the cryptography community is preparing for the quantum threat through new mathematical approaches, standardization, and migration.

---

## 1. The Problem PQC Solves

Today's widely-used public-key cryptography relies on the hardness of:

- **Integer factorization** (RSA)
- **Discrete logarithm / elliptic curve discrete log** (Diffie-Hellman, ECDSA)

**Shor's algorithm**, run on a sufficiently powerful quantum computer, can solve both of these problems in polynomial time — breaking these schemes entirely.

PQC replaces these schemes with algorithms based on mathematical problems believed to be hard for *both* classical and quantum computers.

---

## 2. The PQC Algorithm Families

| Family | Hard Problem | Examples | Status |
|---|---|---|---|
| **Lattice-based** | Shortest vector / learning with errors (LWE) | ML-KEM (Kyber), ML-DSA (Dilithium) | NIST standardized (2024) |
| **Hash-based** | Security of hash functions | SPHINCS+, XMSS, LMS | NIST standardized (2024) |
| **Code-based** | Decoding random linear codes | Classic McEliece, BIKE, HQC | HQC selected as backup KEM (2025) |
| **Multivariate** | Solving systems of multivariate polynomial equations | (largely broken candidates) | Less favored |
| **Isogeny-based** | Maps between elliptic curves | SIDH/SIKE | Broken in 2022 |

### Key Tradeoffs vs. Classical Crypto

- **Larger keys and signatures** — PQC parameters are typically much bigger than RSA/ECC equivalents.
- **Different performance profiles** — some are faster at signing but slower at verification, or vice versa.
- **Less cryptanalytic maturity** — these problems haven't been studied as long as factoring/discrete log.

---

## 3. Lattice-Based Cryptography (Deep Dive)

### What is a Lattice?

A **lattice** is a regular grid of points in n-dimensional space, formed by all integer linear combinations of a set of basis vectors. Think of it like a grid of dots extending infinitely, but in potentially hundreds or thousands of dimensions.

### The Hard Problems

Two core problems underpin lattice crypto:

- **Shortest Vector Problem (SVP):** Given a lattice, find the shortest non-zero vector. In high dimensions, this becomes computationally intractable — even for quantum computers.
- **Learning With Errors (LWE):** Given a system of approximate linear equations (with small random "errors" added), recover the secret. The errors are what make it hard — without them, it's just linear algebra.

There are also **Module-LWE** and **Ring-LWE**, which add algebraic structure to LWE for efficiency (smaller keys, faster operations) at the cost of relying on a slightly stronger assumption.

### How It Becomes Crypto

- **Key Exchange / Encryption (ML-KEM / Kyber):** Alice and Bob exchange noisy lattice-based values. They can approximately agree on a shared secret because they know the secret structure, but an eavesdropper sees only the noisy public data — which looks random without the private key.
- **Digital Signatures (ML-DSA / Dilithium):** Uses a "Fiat-Shamir with Aborts" technique. The signer creates a lattice-based commitment, is challenged, and responds — but *aborts and retries* if the response would leak information about the secret key. This rejection sampling is key to security.

### Why Lattices Won the NIST Competition

| Property | Lattice Schemes | Alternatives |
|---|---|---|
| Key size | ~1 KB (manageable) | Code-based: ~250 KB+ |
| Performance | Fast on commodity hardware | Hash-based sigs: slower verification |
| Versatility | Encryption + signatures + advanced (FHE, ABE) | Hash-based: signatures only |
| Security confidence | Strong theoretical foundations | Comparable, but lattices more studied recently |

### The Risk

Lattice schemes rely on the assumed hardness of structured lattice problems. If a breakthrough algorithm appears (classical or quantum) for Module-LWE, these schemes could fall. This is why NIST also standardized a conservative hash-based alternative.

---

## 4. The NIST PQC Standards (Deep Dive)

### Background

NIST began its **Post-Quantum Cryptography Standardization Process** in 2016, anticipating that standardizing new algorithms would take years and should happen well before quantum computers arrive.

### The Process

- **Round 1 (2017):** 69 submissions across all families
- **Round 2 (2019):** Narrowed to 26 candidates
- **Round 3 (2020):** 7 finalists + 8 alternates
- **Standards published (2024):** Three algorithms standardized as FIPS standards

### The Standards

#### FIPS 203: ML-KEM (based on CRYSTALS-Kyber)

- **Purpose:** Key Encapsulation Mechanism (replaces RSA key exchange, ECDH)
- **Family:** Lattice-based (Module-LWE)
- **How it works:** One party encapsulates a shared secret using the other's public key; only the private key holder can decapsulate it
- **Parameter sets:**
  - ML-KEM-512: ~NIST security level 1 (comparable to AES-128)
  - ML-KEM-768: level 3 (AES-192)
  - ML-KEM-1024: level 5 (AES-256)
- **Key sizes:** Public key ~800-1568 bytes, ciphertext ~768-1568 bytes
- **Performance:** Very fast — encapsulation/decapsulation in microseconds

#### FIPS 204: ML-DSA (based on CRYSTALS-Dilithium)

- **Purpose:** Digital signatures (replaces RSA signatures, ECDSA)
- **Family:** Lattice-based (Module-LWE + Module-SIS)
- **Parameter sets:** ML-DSA-44, ML-DSA-65, ML-DSA-87 (security levels 2, 3, 5)
- **Signature sizes:** ~2.4-4.6 KB (much larger than ECDSA's ~64 bytes)
- **Performance:** Fast signing and verification

#### FIPS 205: SLH-DSA (based on SPHINCS+)

- **Purpose:** Digital signatures (conservative backup)
- **Family:** Hash-based (stateless)
- **Why it exists:** If lattice assumptions break, this still stands — its security depends *only* on the security of the underlying hash function
- **Tradeoff:** Much larger signatures (~7-49 KB) and slower than ML-DSA
- **Use case:** High-assurance environments, or as a hedge against lattice breakthroughs

### Still In Progress

- **HQC** (code-based KEM) was selected in 2025 as a backup KEM to ML-KEM — providing algorithm diversity in case lattice-based KEMs are broken
- **Additional signature schemes** are being evaluated in a separate process for use cases needing shorter signatures (e.g., FALCON-based schemes)

---

## 5. Migration Strategies (Deep Dive)

### Why Migration is Hard

This isn't just "swap one algorithm for another." PQC migration touches:

- Every TLS connection, VPN tunnel, and SSH session
- Certificate chains and PKI hierarchies
- Code signing, firmware updates, secure boot
- Stored encrypted data, key management systems
- Hardware tokens, HSMs, smart cards
- Protocols with hardcoded assumptions about key/signature sizes

### The "Harvest Now, Decrypt Later" Urgency

Data encrypted today with classical algorithms can be recorded and stored. Once a cryptographically relevant quantum computer (CRQC) exists, that data can be decrypted. For data with long confidentiality requirements (medical records, state secrets, financial data), **migration needed to start yesterday**.

### Migration Approaches

#### Hybrid Mode (Recommended Transition)

Combine a classical algorithm with a PQC algorithm so that security holds if *either one* is secure:

```
shared_secret = KDF(ECDH_secret || ML-KEM_secret)
signature_valid = ECDSA_verify(msg) AND ML-DSA_verify(msg)
```

**Why hybrid:**
- PQC algorithms are newer and less battle-tested
- Protects against both quantum attacks *and* potential PQC algorithm breaks
- Allows gradual rollout and fallback

This is what Chrome, Cloudflare, and Signal have already deployed (e.g., X25519 + ML-KEM-768 in TLS).

#### Phased Migration

| Phase | Action | Priority |
|---|---|---|
| **1. Inventory** | Catalog all cryptographic usage — libraries, protocols, key types, data sensitivity | Immediate |
| **2. Protect data-in-transit** | Deploy hybrid KEM in TLS, VPN, messaging | High (harvest-now threat) |
| **3. Update signatures** | Migrate signing keys — certificates, code signing, firmware | Medium |
| **4. Update data-at-rest** | Re-encrypt stored data with PQC-safe keys | Based on data lifetime |
| **5. Retire classical-only paths** | Remove non-hybrid fallbacks once ecosystem matures | Long-term |

#### Crypto Agility

The most important architectural principle: **design systems so cryptographic algorithms can be swapped without rewriting the application.** This means:

- Algorithm identifiers in protocols (not hardcoded choices)
- Abstraction layers around crypto operations
- Key/certificate formats that support new algorithms
- Configuration-driven algorithm selection

Organizations that hardcoded "use RSA-2048 everywhere" are now paying the price. The lesson: assume your crypto *will* need to change again.

### Real-World Migration Already Happening

- **TLS 1.3:** Chrome and Firefox support X25519Kyber768 (hybrid key exchange) by default
- **Signal Protocol:** Upgraded to PQXDH (hybrid X25519 + ML-KEM-1024) in 2023
- **Apple iMessage:** PQ3 protocol with ML-KEM-768 hybrid, deployed 2024
- **SSH:** OpenSSH added hybrid key exchange (sntrup761 + X25519) starting in version 9.0
- **US Government:** NSA's CNSA 2.0 sets deadlines — PQC required for national security systems by 2030-2035

---

## 6. How This Connects to BB84

BB84 (quantum key distribution) and PQC represent **two parallel responses** to the quantum threat:

| | QKD (BB84) | PQC |
|---|---|---|
| **Security basis** | Laws of physics | Mathematical hardness assumptions |
| **Requires special hardware?** | Yes (quantum channel) | No (runs on existing hardware) |
| **Broken by quantum computers?** | No | Believed not to be |
| **Deployable at internet scale?** | Not yet | Yes — already deploying |
| **Provides** | Key distribution only | Encryption, signatures, key exchange |

PQC is the pragmatic, deployable-now solution. QKD is the theoretically stronger but hardware-constrained alternative. Both exist because Shor's algorithm made the status quo untenable.

---

## Timeline

- **1984**: Bennett & Brassard publish BB84
- **1994**: Peter Shor publishes his quantum factoring algorithm
- **2016**: NIST begins post-quantum cryptography standardization
- **2017**: 69 PQC candidates submitted to NIST
- **2022**: SIKE/SIDH (isogeny-based) broken by classical attack
- **2023**: Signal deploys PQXDH hybrid protocol
- **2024**: NIST publishes FIPS 203 (ML-KEM), FIPS 204 (ML-DSA), FIPS 205 (SLH-DSA)
- **2025**: HQC selected as backup KEM
- **2030-2035**: NSA CNSA 2.0 deadlines for PQC adoption in national security systems
