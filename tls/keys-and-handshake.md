# TLS 1.3: Keys, Handshake, and Authentication

## All the Keys in TLS

TLS uses many different keys at different stages. Here's every key involved,
grouped by purpose.

### 1. Long-lived Identity Keys (exist before any connection)

These live in certificates and are used for **authentication**, not encryption.

| Key | Who holds it | Purpose |
|-----|-------------|---------|
| **Server's long-term private key** | Server (kept secret on disk) | Signs the handshake to prove server identity |
| **Server's long-term public key** | Anyone (embedded in the server's X.509 certificate) | Client uses this to verify the server's signature |
| **CA's private key** | Certificate Authority (offline/HSM) | Signs server certificates to vouch for their identity |
| **CA's public key** | Everyone (pre-installed in browsers/OS trust stores) | Verifies that a server's certificate was issued by a trusted CA |
| **Client's long-term private key** | Client (optional, for mutual TLS / mTLS) | Signs the handshake to prove client identity |
| **Client's long-term public key** | Server (in client's certificate, for mTLS) | Server uses this to verify client's signature |

**Important:** These keys are NEVER used for encryption. They only sign and
verify. This is a common misconception — TLS does not encrypt anything with
certificate keys.

### 2. Ephemeral Key Exchange Keys (fresh every handshake)

These are generated at the start of each connection and thrown away after.
They exist solely to establish a shared secret.

| Key | Who generates it | Lifetime |
|-----|-----------------|----------|
| **Client's ephemeral private key** | Client | Single handshake, then deleted |
| **Client's ephemeral public key** | Client (sent in ClientHello `key_share`) | Single handshake |
| **Server's ephemeral private key** | Server | Single handshake, then deleted |
| **Server's ephemeral public key** | Server (sent in ServerHello `key_share`) | Single handshake |

Both sides combine their own ephemeral private key with the other party's
ephemeral public key to independently compute the same **shared secret**. This
is the Diffie-Hellman (or Kyber/ML-KEM in PQC) key exchange.

Because these keys are fresh every time:
- Compromising the server's long-term private key doesn't let you decrypt past
  sessions (this is **forward secrecy**)
- Replaying old messages doesn't work (different ephemeral keys = different
  shared secret)

### 3. Derived Symmetric Keys (computed from the shared secret)

Once both sides have the shared secret from the key exchange, TLS derives a
tree of symmetric keys using HKDF (HMAC-based Key Derivation Function). None of
these are sent over the wire — both sides compute them independently.

```
Shared Secret (from key exchange)
    │
    ▼
Early Secret ──────► (used for 0-RTT data, if applicable)
    │
    ▼
Handshake Secret
    ├──► Client Handshake Traffic Key  ── encrypts client handshake messages
    ├──► Server Handshake Traffic Key  ── encrypts server handshake messages
    │
    ▼
Master Secret
    ├──► Client Application Traffic Key  ── encrypts client application data
    ├──► Server Application Traffic Key  ── encrypts server application data
    └──► Resumption Master Secret        ── used to create session tickets
```

Each "traffic key" actually expands into two values:
- A **symmetric encryption key** (e.g., 256-bit AES-GCM key)
- An **IV / nonce** (initialization vector for the AEAD cipher)

**Why separate keys for client and server?** So traffic in each direction uses
a different key. If client and server used the same key, an attacker could
reflect a message back to the sender and it would decrypt successfully.

**Why separate keys for handshake and application?** Handshake keys encrypt
the certificate exchange. Once the handshake completes, both sides switch to
application traffic keys. This limits exposure — even if handshake keys were
somehow compromised, application data remains protected.

### Summary: Key Count in a Typical Handshake

| Category | Number of keys | Used for |
|----------|---------------|----------|
| Long-term identity keys | 2 (server) + 2 (CA) + 2 (client, if mTLS) | Authentication |
| Ephemeral key exchange keys | 2 client + 2 server = 4 | Deriving shared secret |
| Derived symmetric keys | ~6-8 (handshake + application + resumption) | Encryption |
| **Total** | **~14-16 key values** | |

---

## The TLS 1.3 Handshake, Step by Step

### Overview

```
Client                                              Server
  │                                                    │
  │──── ClientHello ──────────────────────────────────►│
  │     (random, supported ciphers, key_share)         │
  │                                                    │
  │◄─── ServerHello ──────────────────────────────────│
  │     (random, chosen cipher, key_share)             │
  │                                                    │
  │     ┌─────── encrypted with handshake keys ───────┐│
  │◄────│ EncryptedExtensions                         ││
  │◄────│ CertificateRequest (if mTLS)                ││
  │◄────│ Certificate (server's cert chain)           ││
  │◄────│ CertificateVerify (signature over transcript)│
  │◄────│ Finished (MAC over transcript)              ││
  │     └─────────────────────────────────────────────┘│
  │                                                    │
  │     ┌─────── encrypted with handshake keys ───────┐│
  │────►│ Certificate (client's cert, if mTLS)        ││
  │────►│ CertificateVerify (client sig, if mTLS)     ││
  │────►│ Finished (MAC over transcript)              ││
  │     └─────────────────────────────────────────────┘│
  │                                                    │
  │◄═══════ encrypted application data ═══════════════►│
  │         (using application traffic keys)           │
  │                                                    │
```

### Step-by-step

**Step 1: ClientHello** (plaintext)

The client sends:
- `client_random`: 32 bytes of randomness (freshness guarantee)
- Supported cipher suites (e.g., TLS_AES_256_GCM_SHA384)
- Supported key exchange groups (e.g., x25519, secp256r1, ML-KEM-768)
- `key_share`: the client's ephemeral public key(s) for its preferred group(s)
- SNI (Server Name Indication): which hostname the client wants to reach

**Step 2: ServerHello** (plaintext)

The server sends:
- `server_random`: 32 bytes of randomness
- Chosen cipher suite
- `key_share`: the server's ephemeral public key for the chosen group

At this point, both sides independently compute:
1. The **shared secret** from the key exchange
2. The **handshake traffic keys** via HKDF

**Everything after ServerHello is encrypted** with the handshake traffic keys.

**Step 3: EncryptedExtensions** (encrypted)

Server sends extensions that don't need to be in the plaintext ServerHello
(e.g., ALPN protocol selection).

**Step 4: CertificateRequest** (encrypted, optional)

If the server wants mutual TLS (client authentication), it sends this message
telling the client to present a certificate.

**Step 5: Certificate** (encrypted)

The server sends its certificate chain:
```
Server's leaf certificate (contains server's long-term public key)
    └── Intermediate CA certificate
            └── Root CA certificate (client already has this in trust store)
```

**Step 6: CertificateVerify** (encrypted)

The server signs a hash of the entire handshake transcript so far using its
**long-term private key**. This is the critical authentication step:
- It proves the server holds the private key matching the certificate
- It binds the server's identity to THIS specific handshake (because the
  transcript includes both randoms and both key shares)
- An attacker who replays old messages can't forge this signature because the
  transcript will be different

**Step 7: Server Finished** (encrypted)

The server sends a MAC (message authentication code) over the entire handshake
transcript using a key derived from the handshake secret. This proves the
server saw the same handshake messages the client did (integrity check).

**Step 8: Client Certificate + CertificateVerify** (encrypted, only if mTLS)

If the server requested client authentication in step 4, the client now:
1. Sends its certificate chain
2. Signs the handshake transcript with its long-term private key

This is identical to what the server did, but in the other direction.

**Step 9: Client Finished** (encrypted)

The client sends its own Finished MAC. Both sides now:
1. Derive the **application traffic keys** from the master secret
2. Switch to using those keys for all subsequent data

**Step 10: Application data** (encrypted)

HTTP requests and responses flow, encrypted with application traffic keys.

---

## How Authentication Works

### Server Authentication (always happens)

The client needs to answer: "Am I really talking to example.com?"

```
1. Server presents certificate containing its public key
       │
       ▼
2. Client checks certificate validity:
   - Is it expired?
   - Is the domain name (SAN) correct?
   - Is it signed by a trusted CA?
   - Is it revoked? (OCSP / CRL check)
       │
       ▼
3. Client verifies CertificateVerify:
   - Uses the public key FROM the certificate
   - Verifies the signature over the handshake transcript
   - This proves the server has the corresponding private key
       │
       ▼
4. Client verifies Finished MAC:
   - Confirms handshake integrity (no tampering)
```

**What each check prevents:**

| Check | Attack prevented |
|-------|-----------------|
| CA signature on certificate | Attacker can't create fake certificates |
| Domain name in certificate | Certificate for attacker.com can't be used for example.com |
| Expiration / revocation | Stolen or compromised certificates can be invalidated |
| CertificateVerify signature | Attacker who somehow gets the certificate file (without private key) can't impersonate the server |
| Transcript binding | Replay and man-in-the-middle attacks (signature covers THIS handshake's unique randoms and key shares) |

### Client Authentication (mTLS, optional)

Sometimes the server needs to verify the client's identity too. Common in:
- Service-to-service communication (microservices, zero-trust networks)
- Corporate VPNs
- Banking and government systems

The process mirrors server authentication:

```
1. Server sends CertificateRequest
       │
       ▼
2. Client sends its certificate + CertificateVerify (signature over transcript)
       │
       ▼
3. Server validates:
   - Is the client certificate signed by a CA the server trusts?
   - Is the CertificateVerify signature valid?
   - Does the client meet authorization rules? (e.g., correct OU, SAN, etc.)
```

The server's trust store for client certificates is typically separate from the
browser trust store. A company might run its own internal CA specifically for
issuing client certificates.

### Why Signatures Bind to the Session

A subtle but critical point: the CertificateVerify signature covers a hash of
the ENTIRE handshake transcript up to that point. The transcript includes:

- `client_random` (unique per connection)
- `server_random` (unique per connection)
- Both ephemeral key shares (unique per connection)
- All other handshake messages

This means:
1. **Replay impossible**: Old signatures don't match new transcripts
2. **MITM impossible**: An attacker running separate handshakes with client and
   server gets different transcripts, so they can't relay signatures
3. **Downgrade impossible**: If an attacker modifies the ClientHello to remove
   strong cipher suites, the transcript changes and the signature won't verify

---

## How This Connects to Post-Quantum Cryptography

| TLS component | Classical algorithm | PQC replacement | Migration urgency |
|--------------|-------------------|-----------------|-------------------|
| Key exchange | X25519, ECDH P-256 | ML-KEM (Kyber) | **High** — harvest-now-decrypt-later |
| Server signature | ECDSA, RSA-PSS, Ed25519 | ML-DSA (Dilithium) | Lower — requires real-time attack |
| CA signature on certs | ECDSA, RSA | ML-DSA (Dilithium) | Medium — certs have long validity periods |
| Certificate size | ~1-2 KB | ~5-10 KB with ML-DSA | Practical challenge (larger handshakes) |

The key exchange is urgent because recorded traffic is vulnerable today. The
signatures only need to be migrated before quantum computers capable of
breaking ECDSA/RSA actually exist — but given migration timelines, starting
early is still important.
