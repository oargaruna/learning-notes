# Quiz: TLS Keys, Handshake, and Authentication

Date: 2026-04-04

## Section 1: Key Types and Purposes (1/5)

**Q1. Are certificate keys (server's long-term public/private key) ever used for encryption in TLS 1.3?**

My answer: No. They are used only for authentication - e.g. verifying if the server is actually google.com

Result: Correct ✓

---

**Q2. What provides forward secrecy in TLS 1.3, and what does it protect against?**

My answer: Session keys provide forward secrecy. They protect against eavesdropper attacks.

Result: Partially correct ✗ — It's specifically the **ephemeral key exchange keys** (not "session keys"). The specific protection is: if the server's long-term private key is compromised later, past recorded sessions can't be decrypted because the ephemeral keys were deleted after the handshake.

---

**Q3. Why do client and server use separate symmetric keys for each direction of traffic?**

My answer: Ensures that if a session key is leaked, only traffic from one direction is compromised.

Result: Incorrect ✗ — The actual reason is **reflection attacks**: if both sides used the same key, an attacker could replay a message back to the sender and it would decrypt successfully. Separate keys per direction prevent this.

---

**Q4. Why are there separate keys for handshake traffic and application traffic?**

My answer: Both keys perform different purposes. Session keys are only used to encrypt the traffic on the connection. Application keys are used to sign the payload to prove authenticity.

Result: Incorrect ✗ — Both sets are symmetric encryption keys, neither is used for signing. The reason is **limiting exposure**: if handshake keys were compromised, application data remains protected behind different keys.

---

**Q5. How do both sides derive the same symmetric keys without ever sending them over the wire?**

My answer: Using Diffie-Hellman key exchange.

Result: Partially correct ✗ — DH produces the shared secret, but the full answer includes **HKDF** (deterministic key derivation function) which takes the shared secret + **handshake transcript hash** + **fixed labels** from the spec to derive identical keys on both sides.

---

## Section 2: The Handshake (4/5)

**Q6. At what point in the handshake does encryption begin? What keys are used?**

My answer: From the point the server sends EncryptedExtensions. Handshake keys are used to encrypt the traffic.

Result: Correct ✓

---

**Q7. The ClientHello includes a key_share extension. What does it contain and why is it sent so early?**

My answer: I don't know.

Result: Skipped — It contains the client's ephemeral public key(s). Sent early so the server can compute the shared secret immediately after ServerHello, enabling 1-RTT handshake.

---

**Q8. What is the purpose of the CertificateVerify message? How does it differ from just sending a certificate?**

My answer: CertificateVerify confirms that the server owns the long term private key. Certificate is publicly available, and is not sufficient to prove that a server actually is, for example, google.com.

Result: Correct ✓

---

**Q9. What is the purpose of the Finished message (MAC over transcript)?**

My answer: The Finished message proves that the server or client saw the same sequence of messages.

Result: Correct ✓

---

**Q10. Put these handshake messages in order: Finished (server), CertificateVerify, ServerHello, ClientHello, Certificate, EncryptedExtensions**

My answer: ClientHello, ServerHello, EncryptedExtensions, Certificate, CertificateVerify, Finished (server)

Result: Correct ✓

---

## Section 3: Authentication & Attacks (1.5/4)

**Q11. An attacker records a full TLS session and later steals the server's long-term private key. Can they decrypt the recorded session? Why or why not?**

My answer: No. The session is encrypted using ephemeral application keys.

Result: Partially correct ~ — Right conclusion but imprecise. The keys are derived from ephemeral key exchange keys that were deleted. The long-term key is only for signing, not encryption. This is forward secrecy.

---

**Q12. Why can't a man-in-the-middle attacker relay the CertificateVerify signature between two separate handshakes (one with the client, one with the real server)?**

My answer: I don't know.

Result: Skipped — CertificateVerify signs the entire handshake transcript hash, which includes client_random, server_random, and both ephemeral key shares — all unique per connection. Two separate handshakes produce different transcripts.

---

**Q13. In mTLS, what is the server's trust store for client certificates, and how does it relate to the browser's trust store?**

My answer: I don't know.

Result: Skipped — They're typically separate. A company might run its own internal CA for client certificates, independent of the public CA trust store.

---

**Q14. Why is migrating the key exchange to post-quantum algorithms more urgent than migrating signatures?**

My answer: Because of harvest-now-decrypt-later attack. Attacker will record sessions now and decrypt them later when a powerful quantum computer is available.

Result: Correct ✓

---

## Bonus: Tricky Conceptual Questions (0/2)

**Q15. Approximately how many distinct key values are involved in a typical (non-mTLS) TLS 1.3 handshake?**

My answer: Four. Handshake keys, Application keys, Server private keys, master key.

Result: Incorrect ✗ — The answer is ~12-14 distinct values: 2 server long-term + 2 CA + 4 ephemeral + ~6-8 derived symmetric keys.

---

**Q16. If an attacker modifies the ClientHello to remove strong cipher suites (downgrade attack), what mechanism detects this?**

My answer: I don't know.

Result: Skipped — The CertificateVerify signature and Finished MAC both cover the transcript hash, which includes the original ClientHello. Modified ClientHello = different transcript = verification fails.

---

## Summary

| Section | Score |
|---------|-------|
| Section 1: Key Types and Purposes | 1/5 |
| Section 2: The Handshake | 4/5 |
| Section 3: Authentication & Attacks | 1.5/4 |
| Bonus | 0/2 |

### Strengths
- Handshake flow and message ordering
- Authentication concepts (CertificateVerify, Finished)

### Areas to review
- **Transcript binding** — answers Q12 and Q16; the transcript hash ties everything together
- **Key granularity** — distinguish individual key values (public/private pairs, key + IV)
- **Forward secrecy mechanics** — ephemeral keys deleted + long-term keys never used for encryption
