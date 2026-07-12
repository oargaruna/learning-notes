# DirtyClone (CVE-2026-43503) — "write to a file you can only read"

Source: [thehackernews.com — New DirtyClone Linux kernel flaw](https://thehackernews.com/2026/06/new-dirtyclone-linux-kernel-flaw-lets.html)

DirtyClone is the newest member of the same family as **Dirty COW** and **Dirty Pipe**: a kernel flaw that lets an unprivileged process write into file-backed memory it should only be able to read, bypassing file permissions entirely. The vehicle this time is the networking stack's zero-copy path.

---

## The mechanism it abuses: zero-copy skb fragments

When the kernel sends file data over a socket (e.g. `sendfile()`, `MSG_ZEROCOPY`), it avoids copying the bytes. Instead, a socket buffer (skb) holds fragments that point directly at the page-cache pages backing the file on disk. Those pages are shared between the file and the packet.

Because that memory is shared with a file, the kernel tags the fragment with a shared-frag safety bit (the article's `SKB_FRAG_FORKED`). The bit means:

> This fragment's page is shared with a file — you may not write into it in place. If you need to modify it, clone/copy it first (copy-on-write).

That single bit is the whole guardrail. As long as it rides along with the fragment, any code that wants to mutate the packet data will duplicate the page first and leave the file untouched.

## The bug: the safety bit gets dropped when fragments move

Several helper functions that move or copy skb fragments between buffers — `__pskb_copy_fclone()`, `skb_shift()`, and in earlier variants `skb_try_coalesce()` — copy the fragment reference but fail to carry the shared-frag bit across. The article's summary:

> When the kernel copies a network packet internally, two helper functions drop a safety flag that marks the packet's memory as shared with a file on disk.

So after the copy, the destination skb has a fragment still pointing at file-backed page-cache memory, but now stripped of the "don't write in place" mark. The kernel now believes it owns a private, writable buffer — when it's actually looking at the on-disk contents of somebody's file.

## Turning the flag-loss into a root exploit

The attacker needs a code path that (a) is allowed to write into packet data in place, and (b) lets them control the bytes written. In-place IPsec ESP decryption is perfect: the "plaintext" written back into the buffer is whatever the attacker's ciphertext + keys produce.

1. **Wire the target pages in** — memory-map a privileged, world-readable binary like `/usr/bin/su` and get its page-cache pages attached as skb fragments.
2. **Force an internal clone** — push the packet through a helper (`__pskb_copy_fclone()` etc.) that drops the shared-frag bit.
3. **Decrypt in place** — send the packet through an attacker-controlled loopback IPsec tunnel. During in-place ESP decryption the kernel writes the "decrypted" bytes straight into the fragment page — which is really the page cache of `/usr/bin/su`.
4. **Corrupt the auth check** — the attacker chooses those bytes to overwrite su's permission/login check with logic that always succeeds.
5. **Run su** — next execution grants a root shell.

**Net effect:** an arbitrary write into file-backed page cache, needing only read access to the target file. Same outcome as Dirty COW / Dirty Pipe, different subsystem.

## Bug classification — a nuance worth flagging

The article labels it a "write-after-free variant," but that framing is a bit loose. There's no freed object being reused. The precise defect is **lost aliasing/ownership metadata**: a fragment that aliases shared file-backed memory loses the flag that says "shared," so a legitimate in-place write lands on memory the writer doesn't actually own. It's an unintended write primitive from a dropped copy-on-write marker, not a classic UAF or double-free. Mechanically it's much closer to Dirty Pipe (an uninitialized `PIPE_BUF_FLAG_CAN_MERGE` flag letting pipe writes reach page cache) than to a heap-corruption bug.

## Who can exploit it

Setting up the loopback IPsec tunnel needs `CAP_NET_ADMIN`. That sounds privileged, but on distros that enable unprivileged user namespaces by default (Debian, Fedora), a local unprivileged user can spin up a new namespace and gain `CAP_NET_ADMIN` inside it — enough to drive the whole chain. That makes it a local privilege escalation that matters for multi-tenant hosts, CI runners, and Kubernetes nodes.

## The systemic root cause (why it keeps recurring)

DirtyClone is the fourth bug in this class, after **Copy Fail** (CVE-2026-31431), **DirtyFrag** (CVE-2026-43284 / -43500), and **Fragnesia** (CVE-2026-46300). Each prior patch closed one helper that dropped the bit, but the real problem is a contract that isn't enforced anywhere central:

> The underlying problem is not one bad helper function. It is a contract problem: every code path that moves skb fragments has to preserve the shared-frag bit, every time.

That's classic whack-a-mole: the invariant "moving a fragment preserves its shared marking" lives implicitly in dozens of hand-written helpers instead of being enforced by the type system or a single chokepoint, so new violating paths keep surfacing. The durable fix is structural (make it impossible to move a fragment without carrying its metadata), not another one-line flag restoration.

## Mitigations

- **Patch the kernel** — fix merged May 21 (backported to stable/LTS). This is the real fix.
- **Disable unprivileged user namespaces** — `sysctl kernel.unprivileged_userns_clone=0` removes the `CAP_NET_ADMIN`-in-a-namespace foothold.
- **Blacklist `esp4`/`esp6`/`rxrpc` as a stopgap** — removes the in-place-write code paths, but breaks IPsec/AFS, so only for hosts that don't need them.

---

## Deep dive: the two conditions for the write primitive

> "The attacker needs a code path that (a) is allowed to write into packet data in place, and (b) lets them control the bytes written."

This sentence is the hinge between "there's a bug" and "there's an exploit." The dropped flag by itself is just a loaded gun: you have a packet fragment that aliases a file's page-cache memory without the "copy before writing" guard. But a fragment sitting there aliasing a file does no damage until something actually writes to it. Conditions (a) and (b) are the trigger and the aim. Miss either one and the bug is inert.

### (a) "Allowed to write into packet data in place"

The subtlety is the phrase *in place*. Most of the network stack never writes to fragment payload at all, and the paths that do transform data usually allocate a fresh buffer and copy into it. Walk through what happens in each case with your poisoned fragment (which points at `/usr/bin/su`'s page):

- **Send it out a NIC** → the hardware DMA-reads the pages. Read-only. File untouched.
- **Checksum / inspect it** → read-only. File untouched.
- **A transform that copies out** (the common pattern) → kernel allocates new memory, writes the result there, and your file-backed page is only ever a source. File untouched.

None of those corrupt anything, because the write (if any) lands on memory the kernel legitimately owns. To hit the aliased file page, you need a code path that reuses the same physical pages as both input and output — an in-place transform. That's rare and special. In-place-ness is exactly what routes the write onto the file instead of onto a safe scratch buffer.

### (b) "Lets them control the bytes written"

Now suppose you found an in-place write path but you don't control the output bytes. Imagine the kernel zeroed the region in place, or wrote a fixed header. You'd overwrite su's auth check with zeros or garbage — you'd corrupt the binary (maybe crash it), but you couldn't install a working "always return success" bypass. That's a denial-of-service, not privilege escalation.

For an arbitrary-write primitive you need the written bytes to be a function of your input: `output = f(attacker_data)`. Then you can compute the input that produces the exact machine code / bytes you want landing in su's auth check.

### Why IPsec ESP decryption is the perfect fit for both

In-place ESP (Encapsulating Security Payload) decryption satisfies (a) and (b) simultaneously — which is why the exploit reaches for it:

- **(a) In place:** ESP decryption is implemented as an in-place crypto transform to avoid an extra copy — the kernel decrypts ciphertext into the same buffer it read it from. So the "plaintext" output is written straight onto the fragment's pages — which are the file's pages.
- **(b) Fully controlled:** the output is `plaintext = Decrypt(ciphertext, key)`, and the attacker owns both inputs. They craft the ciphertext (they build the ESP packet) and they own the key — they configured the IPsec Security Association themselves via `CAP_NET_ADMIN` on a loopback tunnel, so they're both endpoints. Because decryption under a known key is just an invertible function, they run it backwards: pick the exact bytes they want in su, compute `ciphertext = Encrypt(desired_bytes, key)`, send it, and the kernel "decrypts" it right back to `desired_bytes` and writes them in place.

So decryption effectively becomes "kernel, write these attacker-chosen bytes into this buffer, in place" — precisely the intersection of (a) and (b). That intersection is the whole reason a lost flag turns into root: it's the difference between a fragment that merely aliases a file and a machine that writes arbitrary bytes into that file's page cache on demand.

---

## Deep dive: what in-place ESP decryption actually does

### What ESP is

ESP (Encapsulating Security Payload) is the workhorse protocol of IPsec — the part that actually provides confidentiality. When you set up an IPsec tunnel, outbound packets get their payload encrypted and wrapped in an ESP envelope; inbound packets get that envelope verified and decrypted back to the original data. It's implemented in the kernel's xfrm framework (modules `esp4`/`esp6`).

An ESP-protected packet looks roughly like this:

```
[ IP header ][ ESP header ][      encrypted region      ][ ICV ]
              SPI, seq#      original payload + ESP        auth
                             trailer (padding, next-hdr)   tag
```

- **SPI (Security Parameters Index)** — an ID that tells the receiver which Security Association (SA) — i.e. which key and cipher — to use.
- **Encrypted region** — the original inner packet plus an ESP trailer (padding, pad length, next-header byte).
- **ICV (Integrity Check Value)** — an authentication tag over the packet.

### What ESP decryption does on receive

When an ESP packet arrives, the kernel:

1. Looks up the SA by the SPI in the ESP header → this gives the symmetric key and cipher (e.g. AES-GCM, or AES-CBC + HMAC-SHA256).
2. Verifies the ICV (authentication) so a tampered packet is rejected. With AEAD ciphers like AES-GCM this is combined with step 3.
3. **Decrypts the encrypted region** — runs the cipher in decrypt mode over the ciphertext bytes to recover the plaintext (the inner packet + trailer).
4. Strips the ESP framing using the trailer and hands the recovered inner packet back up the stack.

Step 3 is the one that matters for DirtyClone.

### Why decryption is done "in place"

The kernel's crypto API works over scatterlists — descriptors that say "the data lives in these memory pages at these offsets." An AEAD/decrypt request (`aead_request`) is given a source scatterlist and a destination scatterlist.

For ESP decryption, the encrypted region and the resulting plaintext are the same length (a block cipher's ciphertext and plaintext blocks are equal size). So the kernel doesn't bother allocating a second buffer — it points the source and destination scatterlists at the same skb fragment pages. The cipher reads a ciphertext block out of a page, computes the plaintext, and writes it back to the exact same offset in the same page. Src == dst. That's "in-place decryption."

The motivation is pure performance — the same zero-copy philosophy as the fragment aliasing itself: don't allocate, don't copy, transform the buffer where it already sits. Concretely the crypto walk does something like:

```c
for each block in the encrypted region:
    ciphertext = read(page, offset)       // read from skb fragment page
    plaintext  = cipher_decrypt(key, ciphertext)
    write(page, offset, plaintext)        // write back to the SAME page
```

### Why that's the loaded trigger for the exploit

Normally those fragment pages are the packet's own private scratch memory, so writing plaintext back into them is harmless. But recall the DirtyClone bug: the dropped shared-frag flag left a fragment aliasing `/usr/bin/su`'s page-cache pages without the "copy before writing" guard. Now the in-place `write(page, offset, plaintext)` in that loop lands directly on the file's page cache.

And because the attacker owns the SA (they installed it with `CAP_NET_ADMIN`, e.g. via `ip xfrm state add ...` with a key they chose), they control the cipher key. So they run the cipher backwards: to get `desired_bytes` written into su, they compute `ciphertext = Encrypt(desired_bytes, their_key)`, put it in an ESP packet aimed at the loopback tunnel, and the kernel's in-place decrypt "recovers" `desired_bytes` and writes them straight into the binary's page cache.

So "in-place ESP decryption" is, from the attacker's point of view, a kernel-blessed primitive that says: read these pages, run my chosen key over them, and write my chosen bytes back into the very same pages — exactly the controlled, in-place write that conditions (a) and (b) required.

---

## The reusable pattern

> **Shared page cache + a zero-copy path that aliases it + a lost copy-on-write flag = write to a file you can only read.**

This is the same skeleton as **Dirty COW** and **Dirty Pipe**, just relocated to the networking stack's zero-copy path.