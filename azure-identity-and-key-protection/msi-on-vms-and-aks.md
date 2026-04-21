# Managed Identity on VMs and AKS

Common thread across both: **no long-lived secrets land on the workload**. The platform attests the workload's identity to Entra ID on its behalf.

## On VMs — IMDS

- Token endpoint at the link-local IMDS address `169.254.169.254` (non-routable; reachable only from inside the VM).
- IMDS is served by the **Azure host / hypervisor**, not the guest OS. The host holds a **fabric-issued X.509 certificate** (rotated roughly every 46 days by the Azure fabric controller) and uses it to authenticate the VM's identity claim to Entra ID.
- The guest never sees the certificate. It calls IMDS over plain HTTP and gets back a short-lived (~24h) bearer token.
- **System-assigned** identity is tied to the VM lifecycle. **User-assigned** is a standalone ARM resource bound to the VM via RBAC assignment.

### Hardening on VMs

- Restrict IMDS to root/admin via host firewall (e.g. iptables owner match), so only trusted daemons can read tokens.
- Require the `Metadata: true` header (IMDSv2-style) to resist SSRF.
- Block SSRF paths in any app that proxies outbound requests — classic capture-the-IMDS-token attack.

## On AKS — Workload Identity

Modern mechanism. **Pod Identity / aad-pod-identity is deprecated** (2022).

- Cluster exposes an **OIDC issuer** (public JWKS URL).
- Each pod receives a **projected service account token** (short-lived JWT signed by the cluster) mounted at `/var/run/secrets/azure/tokens/azure-identity-token`.
- In Entra ID, a **federated identity credential** on the target user-assigned managed identity trusts the tuple:
  - `issuer` = cluster OIDC URL
  - `subject` = `system:serviceaccount:<ns>:<sa>`
  - `audience` = `api://AzureADTokenExchange`
- At runtime, the workload (via MSAL / Azure SDK) performs a **client-assertion grant**: sends the K8s SA JWT to Entra, which validates the signature against the OIDC JWKS and issues an Entra access token.
- No secrets, no NMI DaemonSet intercepting IMDS, no iptables rewrite tricks. Pure federation.

### Verification checklist

- OIDC issuer enabled on the cluster (`--enable-oidc-issuer`).
- Workload identity enabled (`--enable-workload-identity`).
- SA annotated with `azure.workload.identity/client-id=<uami-client-id>`.
- Pod labeled `azure.workload.identity/use: "true"`.
- Federated credential's subject matches `system:serviceaccount:<ns>:<sa>` exactly.
