# BB84: The Paper That Started Quantum Cryptography

This is a novice-friendly breakdown of the foundational **BB84 paper** by Charles H. Bennett and Gilles Brassard, originally presented at the International Conference on Computers, Systems and Signal Processing in Bangalore, December 1984. Published in *Theoretical Computer Science* 560 (2014) 7–11.

---

## The Big Idea

**Regular encryption** (like RSA) relies on math problems being *hard* to solve. If someone builds a powerful enough computer, those systems break.

**Quantum cryptography** relies on the *laws of physics* instead. The core insight: **you can't spy on a quantum message without disturbing it**, and that disturbance is detectable.

---

## How It Works: Quantum Key Distribution (QKD)

Think of it like sending secret messages using light particles (photons). The paper describes a protocol with two people, traditionally called **Alice** and **Bob**:

1. **Alice sends photons** to Bob, each encoding a bit (0 or 1). For each photon, she randomly picks one of two "languages" (called **bases**) to encode it — either rectilinear (+) or diagonal (x).

2. **Bob measures each photon**, but he doesn't know which "language" Alice used, so he randomly guesses one for each photon. When he guesses right, he reads the bit correctly. When he guesses wrong, he gets random garbage.

3. **They compare notes publicly.** Over a normal channel, they tell each other *which bases they used* (but NOT the bit values). They keep only the bits where they happened to use the same basis — roughly half.

4. **They check for spies.** They sacrifice a random subset of their shared bits, comparing them publicly. If the bits match, no eavesdropper was present. If they don't match, someone was listening.

5. **The surviving bits become their secret key**, used as a one-time pad for perfectly secure communication.

---

## Why Eavesdropping Fails

This is the clever part. If an eavesdropper ("Eve") intercepts a photon:

- She doesn't know which basis Alice used.
- She has to guess, and she'll guess wrong ~50% of the time.
- When she guesses wrong, she irreversibly corrupts the photon.
- When Bob and Alice compare their check bits, **they'll see mismatches** and know someone was listening.

The physics guarantee: **you cannot copy a quantum state** (the "no-cloning theorem"), and **you cannot measure it without changing it** (the uncertainty principle).

### Eavesdropping Trade-off

The paper proves a precise bound: no measurement by an eavesdropper (who learns the photon's original basis only after measuring) can yield more than 1/2 expected bits of information per photon. Any measurement yielding *b* bits of expected information must induce a disagreement with probability at least *b/2* when the photon is later re-measured in its original basis.

---

## Worked Example (from the paper)

```
QUANTUM TRANSMISSION
Alice's random bits:          0  1  1  0  1  1  0  0  1  0  1  1  0  0  1
Random sending bases:         D  R  D  R  R  R  R  R  D  D  R  D  D  D  R
Random receiving bases:       R  D  D  R  R  D  D  R  D  R  D  D  D  D  R

PUBLIC DISCUSSION
Bob reports bases of received bits
Alice says which bases were correct (marked OK)
Presumably shared information (if no eavesdrop): 1  1  0  1  0  1

Bob reveals some key bits at random:  1  0
Alice confirms them:                  OK OK

OUTCOME
Remaining shared secret bits: 1  0  1  1
```

Key: R = Rectilinear basis, D = Diagonal basis

---

## The Second Protocol: Quantum Coin Tossing

The paper also describes a way for two *distrustful* parties to flip a fair coin remotely:

1. Alice picks a basis and sends random photons in that basis to Bob.
2. Bob measures them in random bases and guesses which basis Alice used.
3. Alice reveals the answer — Bob can verify she's honest by checking his measurement tables.

### The EPR Loophole

Alice *can* cheat using the Einstein-Podolsky-Rosen (EPR) effect — sending entangled photon pairs instead of regular photons, then measuring her stored half after Bob's guess. This gives her results perfectly correlated with Bob's table for whichever basis she claims.

However, this requires **perfect photon storage and detection efficiency**, which is impractical with current (or foreseeable) technology.

---

## Glossary of Key Concepts

| Concept | Plain English |
|---|---|
| **Quantum channel** | A fiber or free-space link carrying single photons |
| **Classical channel** | A normal communication line (phone, internet) |
| **Basis** | The "language" used to encode/decode a photon — rectilinear (horizontal/vertical) or diagonal (45/135 degrees) |
| **Conjugate bases** | Two bases where knowing a state in one basis gives completely random results in the other |
| **One-time pad** | An unbreakable encryption method where the key is as long as the message and used only once |
| **No-cloning theorem** | You can't make a perfect copy of an unknown quantum state |
| **EPR effect** | Two entangled particles are mysteriously correlated, no matter how far apart |
| **Hilbert space** | The mathematical space used to describe quantum states (2-dimensional for a single photon's polarization) |
| **Wegman-Carter authentication** | A method for verifying message integrity using a small shared secret key |

---

## Why This Paper Matters

- It proved that **physics can guarantee security**, not just math.
- It's secure against attackers with **unlimited computing power** (including future quantum computers).
- It launched the entire field of quantum cryptography.
- Real QKD systems based on BB84 are commercially available today.

## Where Shor's Algorithm Fits In

Shor's algorithm isn't mentioned in this paper — it was published by Peter Shor in **1994**, a full decade after BB84. But it's the reason BB84 became so critically important.

### The Threat

Shor's algorithm is a quantum computing algorithm that can **efficiently factor large numbers** and **compute discrete logarithms**. This directly breaks:

- **RSA** (relies on factoring being hard)
- **Diffie-Hellman** (relies on discrete logarithm being hard)
- **Elliptic Curve Cryptography** (relies on elliptic curve discrete logarithm being hard)

These are the systems that protect virtually all internet traffic today — HTTPS, banking, messaging, etc.

### Why BB84 Is Immune

The BB84 paper says something prescient in its introduction: the theory of computational complexity is *"not yet well enough understood to prove the computational security of public-key cryptosystems."* They were right to worry.

| | Traditional Crypto (RSA, etc.) | BB84 / QKD |
|---|---|---|
| **Security basis** | Math problems assumed to be hard | Laws of physics |
| **Broken by Shor's algorithm?** | Yes | No |
| **Broken by unlimited computing power?** | Yes | No |

BB84 is **immune to Shor's algorithm** because its security doesn't depend on any computational assumption at all. Even an attacker running Shor's algorithm on a perfect quantum computer gains nothing — the security comes from the no-cloning theorem and the uncertainty principle, which are fundamental physical laws.

### The Two Responses to Shor's Algorithm

The cryptography community has pursued two parallel tracks:

1. **Quantum Key Distribution (QKD)** — the BB84 approach. Use quantum physics to distribute keys securely. Requires specialized hardware (photon sources, quantum channels).

2. **Post-Quantum Cryptography (PQC)** — design new classical algorithms based on math problems that are hard even for quantum computers (e.g., lattice-based, hash-based, code-based cryptography). This is what NIST has been standardizing since 2016.

### Timeline

- **1984**: Bennett & Brassard publish BB84 (this paper)
- **1994**: Peter Shor publishes his quantum factoring algorithm
- **2016**: NIST begins post-quantum cryptography standardization
- **2024**: NIST publishes first PQC standards (ML-KEM, ML-DSA, SLH-DSA)

In short: Shor's algorithm is the **attack** that makes traditional cryptography vulnerable. BB84 is one of the **defenses** — proposed a decade before the threat even existed.

---

## Limitations Noted in the Paper

- **Quantum transmissions are weak** — photons can't be amplified without destroying their quantum properties.
- **Does not provide digital signatures**, certified mail, or dispute resolution.
- The quantum channel only distributes *keys*, not messages directly.
- Practical detectors are imperfect, leading to photon loss.

---

## Deep Dives: Q&A

### What is the "guesswork" mentioned in the introduction?

> "Conventional cryptosystems such as ENIGMA, DES, or even RSA, are based on a mixture of guesswork and mathematics."

The authors are making a pointed observation: the security of traditional cryptosystems rests on **unproven assumptions** — things we *believe* to be true but can't mathematically guarantee:

- **ENIGMA**: Its designers *guessed* that the mechanical complexity of rotor permutations would be enough. It wasn't — Turing and others broke it.
- **DES**: Its design involved S-boxes (substitution tables) partly chosen by the NSA. The broader community had to *trust* that no hidden weaknesses were embedded. The key length (56 bits) was also a judgment call that proved too short.
- **RSA**: It *assumes* that factoring large numbers is computationally hard. Nobody has proven this. It's an educated guess — and Shor's algorithm later showed it *is* easy on a quantum computer.

The "mathematics" part refers to the formal structures these systems use (modular arithmetic, permutation groups, number theory). But the **security claims** linking that math to actual unbreakability are the "guesswork."

Bennett and Brassard are contrasting this with their approach: quantum cryptography's security comes from **proven physical laws** (uncertainty principle, no-cloning theorem), not from hoping that some math problem stays hard.

### What does "the key must be at least as long as the cleartext" mean?

> "Information theory shows that traditional secret-key cryptosystems cannot be totally secure unless the key, used once only, is at least as long as the cleartext."

This is a reference to **Shannon's theorem** (Claude Shannon, 1949). If you want **mathematically perfect secrecy** — meaning an attacker learns *absolutely nothing* about your message, even with infinite computing power — then your secret key must be:

1. **At least as long as the message itself**
2. **Truly random**
3. **Used only once**

**Why this must be true:** Say your message is 100 bits long. If your key is also 100 random bits, every possible 100-bit message is an equally valid decryption — an attacker can't tell which one is correct. But if your key is only 10 bits, there are only 1,024 possible keys, and an attacker can try all of them and narrow down which messages are plausible. Information leaks.

This creates a catch-22: perfect secrecy requires a key as long as the message, but how do you securely deliver a key that long? BB84 breaks this catch-22 — the quantum channel lets Alice and Bob generate an arbitrarily long shared random key at a distance, with physics guaranteeing no one else knows it.

### What is a one-time pad?

A one-time pad is the simplest possible encryption scheme. You combine your message with a random key using XOR (exclusive or), bit by bit:

```
Message:    1 0 1 1 0 0 1 0
Key:        0 1 1 0 1 0 1 1
            ─────────────────
Ciphertext: 1 1 0 1 1 0 0 1
```

To decrypt, the recipient XORs the ciphertext with the same key and gets the original message back.

**Why "one-time":** The key must be used only once. If you reuse it, an attacker can XOR two ciphertexts together — the key cancels out, and patterns from the messages start to emerge.

**Why it's perfectly secure:** Every possible message is equally likely to produce the ciphertext. If an attacker intercepts `1 1 0 1 1 0 0 1`, it could decrypt to *any* 8-bit message depending on the key. There's no way to narrow it down — not with brute force, not with a quantum computer, not with infinite time.

**The classic problem:** It's impractical. To send a 1 GB file securely, you need 1 GB of key material that both parties already share. Historically, this meant physically transporting keys (diplomatic pouches, codebooks on submarines, etc.).

**How BB84 solves it:** The quantum channel lets Alice and Bob continuously generate fresh shared random bits at a distance. They use those bits as one-time pad keys. When the pad runs out, they run the protocol again to generate more.

---

*Source: Bennett, C.H. & Brassard, G. (2014). Quantum cryptography: Public key distribution and coin tossing. Theoretical Computer Science, 560, 7-11. Originally presented in 1984.*
