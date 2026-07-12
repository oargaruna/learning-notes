# Kubernetes node identity — attestation vs. bootstrap-token impersonation

Notes from the WG-Creation-Request: **WG Node Identity** proposal on the
Kubernetes `dev@` list (2026).
Source: https://groups.google.com/a/kubernetes.io/g/dev/c/Blxf5Vu22Kk

Authors: Naadir Jeewa + co-chairs (Rodrigo Campos Catelin, Ciprian Hacman,
Michael McCune, Josephine Pfeiffer). Cross-SIG WG spanning SIG Auth, SIG Node,
SIG Cluster Lifecycle.

## The problem in one line

Bootstrap tokens authenticate *"someone allowed to join a node"* but **never
identify *which machine* is joining** — yet the entire per-node authorization
model silently depends on that identity being real.

> "any bearer of a valid token can register as any node name, and there is no
> mechanism for the control plane to cryptographically verify that a kubelet is
> running on the machine it claims to be."

This is a trust-on-first-use (TOFU) model.

## How node join works today

1. **Token** authenticates the holder as group `system:bootstrappers` (+ any
   configured extra groups). RBAC lets that group do two narrow things: submit a
   CSR, and have node-client CSRs auto-approved. The token says nothing about
   *which* node name is allowed.
2. **CSR** — the kubelet picks its *own* name and puts it in the subject as
   `CN=system:node:<nodename>`, org `system:nodes`. The requester chooses this
   string; the issuer does not verify it.
3. **Cert** — auto-approver
   (`system:certificates.k8s.io:certificatesigningrequests:nodeclient`) approves
   based on the *group*, not on any machine check. Result: valid token + any node
   name → signed `system:node:<name>` cert.

```
token  →  "you may submit a node-client CSR"   (group-level, machine-agnostic)
CSR    →  "...and here's the node name I claim"  (attacker-chosen, unverified)
cert   →  system:node:<whatever-you-asked-for>   (signed, trusted by Node authz)
```

## Why the identity matters: the Node authorizer

The **Node authorizer** + **NodeRestriction** admission plugin scope a
`system:node:nodeA` cert to exactly the objects nodeA needs: Secrets, ConfigMaps,
PVCs/PVs, and node/pod objects for pods scheduled onto nodeA. Good scoping — but
it all assumes `system:node:nodeA` really *is* nodeA. The token model breaks that
assumption.

## The impersonation attacks (what this prevents)

1. **Stolen/leaked token → register as any node.** Token is a bearer secret, not
   bound to a machine. Leaks happen via join commands in CI logs, Terraform
   state, ASG user-data, Slack, etc.
2. **Impersonate a high-value node → steal its secrets.** Register as
   `system:node:nodeA`; the Node authorizer serves all Secrets mounted by pods on
   nodeA (TLS keys, DB creds, SA tokens). Node impersonation ≈ scoped credential
   exfiltration.
3. **Lateral movement between nodes.** A compromised node B assumes node A's
   identity, collapsing per-node isolation.
4. **Pre-register / hijack a node name.** No crypto anchor ties a name to
   hardware, so names can be squatted or re-registered.

## Can an attacker just mint tokens? No.

A bootstrap token is a `Secret` in `kube-system`, type
`bootstrap.kubernetes.io/token`. Format: `[a-z0-9]{6}.[a-z0-9]{16}` — public
6-char **ID** (also the Secret name suffix `bootstrap-token-<id>`) + secret
16-char proof. Metadata holds expiry, allowed usages, and `auth-extra-groups`.

Creating one needs **write access to `kube-system` Secrets** — a privileged op.
The `system:bootstrappers` group a token grants *cannot* create Secrets, so a
token-joined node cannot mint more tokens. An attacker who can write those Secrets
already out-powers the token entirely (can sign certs / read all secrets). So the
risk is **leaked/over-scoped legitimate tokens**, not forgery.

## The fix: hardware-backed attestation

Fold attestation into the CSR flow — **TPM / vTPM keys** or **cloud
instance-identity documents** — so the control plane verifies the requester
actually possesses the hardware mapping to the requested node name. Goal: as
turnkey as TLS bootstrap tokens — on by default where the platform supports it.

## What attestation does and does NOT eliminate

Attestation changes the binding to **node name ↔ hardware identity.** A leaked
token can then only yield a cert for hardware the attacker genuinely possesses.

- **Eliminated outright:** impersonating a *different existing* node, and
  cross-node secret theft (attacker can't produce nodeA's attestation).
- **Residual case 1 — attacker on a compromised existing node:** can get
  `system:node:nodeX`, but they already own nodeX's kubelet/cert/secrets → no
  escalation.
- **Residual case 2 — attacker stands up their own attestable machine:** leaked
  token can enroll it as a *new legitimate* node. But it only gets secrets for
  pods the scheduler places on it (≈ nothing sensitive by default); stealing more
  needs other permissions to attract sensitive pods.

**Key shift:** the problem goes from a *cryptographic identity* problem ("can't
tell who a node is") to an *enrollment-policy* problem ("who may join at all") —
and attestation is what *lets* you solve the latter: the approver can **allowlist**
verified identities (specific ASG/account/project, inventory-enrolled TPMs), so
even a valid leaked token can't enroll an off-list machine. Short/one-time token
TTLs shrink the window further.

## Related
- [[../pod-security-admission]] — other side of node/pod trust boundary (host fields)
- [[../azure-identity-and-key-protection]] — TPM/attestation & instance identity themes
