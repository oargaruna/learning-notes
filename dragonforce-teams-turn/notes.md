# DragonForce — abusing Microsoft Teams TURN relays for C2

Notes from The Hacker News, 2026-06.
Source: https://thehackernews.com/2026/06/dragonforce-hackers-abuse-microsoft.html

## Who

**DragonForce** — ransomware group that shifted from a classic
ransomware-as-a-service (RaaS) model to an organized "cartel" structure.
Activity ran Dec 2025 → early 2026, with dwell time of 1–2 months per victim.

## Background: what is TURN (and STUN / ICE)?

The problem TURN solves is **NAT traversal**. Most devices sit behind NAT
(Network Address Translation) — they have a private LAN IP (e.g. `192.168.x.x`)
and share one public IP via a router/firewall. NAT is great for outbound traffic
but breaks peer-to-peer apps (voice/video, games): an outside peer has no routable
address to send packets *to*, and the firewall drops unsolicited inbound packets.

Three related protocols (collectively the **ICE** framework) fix this:

- **STUN** (Session Traversal Utilities for NAT) — a lightweight "what's my public
  address?" service. A client asks a STUN server, which replies with the public
  IP:port the request appeared to come from. If the NAT is permissive, two peers
  can then talk **directly**. Cheap, but fails with strict/symmetric NATs.
- **TURN** (Traversal Using Relays around NAT) — the fallback when direct P2P is
  impossible. Instead of connecting peer-to-peer, **both peers connect outbound to
  a shared public relay server, which forwards packets between them.** Since both
  connections are outbound-initiated, they sail through NAT/firewalls. Reliable but
  costs the operator bandwidth (all media flows through the relay).
- **ICE** (Interactive Connectivity Establishment) — the decision logic that
  gathers all candidate paths (direct, STUN-discovered, TURN-relayed), tries them
  in order of preference, and picks the best one that works. Direct first, STUN
  next, TURN as last resort.

```
Direct (ideal):     Peer A  ───────────────►  Peer B
STUN-assisted:      Peer A  ──(via discovered public IP)──►  Peer B
TURN-relayed:       Peer A  ──►  TURN relay  ◄──  Peer B   (both outbound)
                                (forwards packets between them)
```

Teams/Skype use this stack for calls. The relay is a **trusted, neutral middleman
that forwards opaque traffic between two parties** — and that's exactly the
property DragonForce abused.

## The headline technique: living off a trusted service

First publicly documented abuse of **Microsoft Teams TURN relays** for C2.

The attackers used the relay **as designed** — as a neutral forwarder — but with
their malware as one "peer" and their C2 server as the other. They didn't
compromise the relay itself; they just borrowed Microsoft's infrastructure as an
intermediary hop to hide their real C2 server.

```
Infected victim machine
      │  outbound — looks like normal Teams traffic
      ▼
Microsoft Teams TURN relay   ← legitimate MS server, abused as a relay
      │  QUIC session forwarded onward
      ▼
Attacker's real C2 server
```

**Why it works:** defenders only see outbound traffic to `*.teams.microsoft.com`
— a trusted, ubiquitous Microsoft domain that's essentially never blocked. The
real C2 stays hidden behind the relay.

To use the relay the backdoor didn't need a stolen account — it grabbed an
**anonymous Teams "visitor" token** (guest-join style access) from Microsoft's
Skype-backed identity services, enough to get the relay to forward its traffic.

This is a flavor of **"living off trusted services"** (a.k.a. domain/protocol
fronting) — hiding malicious traffic inside connections to a reputable provider.

## Attack chain

1. **Initial access** — likely SQL/MS-SQL server exploitation, or bought from an
   initial access broker.
2. **Execution** — PowerShell drops a ZIP disguised as a "tech support hotfix";
   **DLL side-loading** runs a rogue DLL for recon.
3. **Defense evasion (BYOVD — Bring Your Own Vulnerable Driver)** — load signed
   but vulnerable drivers to blind/kill endpoint security from the kernel.
   Drivers seen: Huawei `HWAuidoOs2Ec.sys`, `wsftprm.sys`, `GameDriverX64.sys`,
   `K7RKScan.sys`, `ABYSSWORKER`.
4. **Ransomware deployment.**
5. **Post-ransomware backdoor** — `Backdoor.Turn`.

## Backdoor.Turn

Go-based RAT, injected into a legitimate process (`DbgView64.exe`). Capabilities:
command execution, process creation, network scanning, LDAP/Active Directory
enumeration, credential-based lateral movement, browser credential theft — plus
the TURN-relay C2 channel described above.

## Defensive takeaways

- TURN/Teams traffic to Microsoft IPs is **not inherently trustworthy**. Watch for
  sustained TURN/QUIC sessions from servers that have no reason to make Teams calls.
- Harden against **BYOVD** — Microsoft vulnerable-driver blocklist via WDAC/HVCI.
- Watch for **DLL side-loading** and unexpected PowerShell-dropped archives;
  patch internet-facing MS-SQL.

## Caveats / to verify

Sourced via a page summarizer — confirm exact strings against the original
article / underlying vendor report before citing: driver/process names, the
"Hackledorb" actor alias, and the `Backdoor.Turn` naming.
