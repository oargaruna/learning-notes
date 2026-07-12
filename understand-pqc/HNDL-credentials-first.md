# HNDL Prioritization: Credentials First, Not the HTTPS Channel

Source: The Hacker News, 2026-06 — "Why Post-Quantum Cryptography Starts With..."

**My prior assumption:** securing the HTTPS/TLS channel with PQC is the first
priority against harvest-now-decrypt-later (HNDL).
**Correction:** both need PQC, but if you must *sequence* the migration under a
finite budget, long-lived **credentials/secrets** come first. HTTPS-first
optimizes for *where data moves*; HNDL-first optimizes for *how long the secret
stays dangerous*.

## What HNDL actually threatens
Attacker records ciphertext **today**, stores it, decrypts it when a CRQC exists
(article: possibly "within 15 years"). So the decisive variable is NOT what is
encrypted, but:

> When this ciphertext is cracked (~10–15 yrs out), will the plaintext still be damaging?

Treat "any data intercepted and harvested today ... as data already exposed."

## The conflation to unlearn: channel vs. payload
"Secure HTTPS first" bundles two things:
- **Channel** = TLS key exchange (protects data *in transit*). PQC-ing it (hybrid
  ML-KEM) IS a real HNDL defense — see [[PQC-migration-design]].
- **Payload** = what flows through: page loads, API responses, session tokens, OR
  credentials.

What do you *win* by cracking one harvested TLS session in 15 yrs?
- Mostly transient data + a **session token that expired years ago** → worthless.
  "Session tokens have a confidentiality lifetime measured in months."
- Yield = **one session's** worth of data.

## Why credentials jump the queue — confidentiality-lifetime disparity
"Credentials may persist for years or as long as their associated systems remain
in service." A credential harvested today and **still valid at decryption time**
hands the attacker *live, privileged access* — not a stale snapshot.

Three factors stack toward credentials:
1. **Lifetime:** tokens = months; credentials = years→decades. Only long-lived
   plaintext survives to the decryption date to be worth harvesting.
2. **Blast radius:** one cracked session = one session's data; one cracked
   credential = ongoing pivot into every system it unlocks.
3. **Scale & blindness:** sprawling **Non-Human Identities** (service accounts,
   API keys) "likely have not been inventoried for cryptographic exposure."

## The prioritization rule (the reusable mental model)
Rank by **confidentiality lifetime × reachability/blast radius** — NOT by system
size or "encrypt the biggest pipe."

> "A small, long-lived secret that brokers access to critical systems outweighs a
> vast but short-lived dataset."

| Harvested today | Cracked in ~15 yrs | Consequence |
|---|---|---|
| TLS session (page, token) | stale token / old page | limited — already expired |
| Long-lived credential | still-valid key/secret | **instant privileged access now** |

**Rotation is the lever credentials have that traffic doesn't:** a frequently
rotated secret has a short *effective* confidentiality lifetime. Long-lived,
never-rotated machine identities are the sharpest HNDL exposure.

## Recommended roadmap
1. **Inventory** crypto dependencies where secrets live — password managers,
   secrets managers, PAM/vaults.
2. **Prioritize** by confidentiality lifetime × blast radius.
3. **Adopt hybrid crypto** (classical + quantum-resistant together).
4. **Design for crypto-agility** — algorithm swaps become a *config change, not a
   re-engineering project*; centralize so one update propagates.

Related: [[PQC-migration-design]] (channel-side hybrid KEM), [[PQC-overview]].
