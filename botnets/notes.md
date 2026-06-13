# Botnets

Notes from studying the JDY botnet (The Hacker News, 2026-06).
Source: https://thehackernews.com/2026/06/china-linked-jdy-botnet-expands-to-1500.html

## What is a botnet?

A network of internet-connected devices an attacker has secretly compromised
and controls remotely as one coordinated group.

- **Bot / zombie** — an individual infected device.
- **Botmaster** — the operator controlling them.
- **C2 (command-and-control)** — infrastructure used to issue orders and collect
  results. Often hidden behind Tor, proxies, or peer-to-peer designs so there's
  no single server to seize.

The device's real owner usually has no idea — it keeps working normally while
serving the attacker.

## Lifecycle

1. **Infection** — exploit a vulnerability or weak/default credentials. Edge
   devices (routers, cameras) are favorite targets: always on, rarely patched,
   directly internet-facing.
2. **Payload delivery** — small dropper installs malware matching the device's
   CPU architecture.
3. **Phone home (C2)** — each bot connects back for instructions and reporting.
4. **Execution** — the fleet acts in unison on command.
5. **Persistence & growth** — survives where it can, keeps hunting new victims.

## Common uses

- DDoS attacks (flooding a target offline)
- Spam / phishing distribution
- Credential stuffing / brute-forcing
- Crypto-mining on stolen compute
- **Reconnaissance** — mass-scanning the internet to map vulnerable systems (JDY)

## JDY botnet (concrete example)

- **Size/origin:** 1,500+ devices; China-nexus state-sponsored. Started as a
  cluster inside the older KV-botnet, split off after early-2024 takedowns.
  Linked to Volt Typhoon.
- **Bots:** SOHO routers and IoT gear (Cisco, Araknis, Mimosa, Ubiquiti,
  Draytek, Hikvision, Linksys). Mostly U.S. and Brazil.
- **Infection:** exploits *newly disclosed* edge-device CVEs, then a shell-script
  dropper pulls an architecture-specific payload (mips, mips64, mipsel, mipsel64).
- **C2:** hidden behind Tor nodes.
- **Purpose:** not direct attack — it's a distributed scanner. Discovers,
  fingerprints, and continuously maps exposed services, then feeds that
  intelligence to other Chinese nation-state groups for follow-on exploitation.
  It's the scout, not the assault team.

## The three recon activities

**Exposed service** = a program listening on a port (web 443, SSH 22, DB 3306,
RDP 3389, camera panel...) that is reachable from the public internet. Each is a
potential door.

1. **Discover** — find which IPs have something listening.
   - *SYN scan / raw sockets*: send only the first TCP handshake packet; SYN-ACK
     means open, then drop. Fast and stealthy ("half-open"), needs root.
   - *Standard TCP/TLS*: open a normal full connection. Slower/noisier, no
     special privileges needed. (JDY picks the method based on privileges it has.)
   - Output: "IP X has 443, 22, 8080 open."

2. **Fingerprint** — identify the exact software, version, and config.
   - *Banner grabbing*: services often announce themselves (`Server: Apache/2.4.49`,
     SSH version strings, TLS cert details).
   - *Behavioral probing*: crafted requests matched against signature databases.
   - Matters because vulnerabilities are version-specific. Turns "a door exists"
     into "a model-2.4.49 lock we know how to pick." This is why JDY races to
     scan right after a CVE drops.

3. **Continuously map** — rescan over time to keep a living inventory, because the
   internet constantly changes (devices appear, get patched, get misconfigured).
   When a new exploit drops, operators already have the target list.

**Summary:** Discover = *where are the doors?* · Fingerprint = *what lock, is it
pickable?* · Continuous mapping = *keep the answer current.*

Legitimate equivalents: Shodan, Censys (publicly index exposed services). JDY is
the malicious distributed version using thousands of hijacked routers.

## Detecting infection

### Hard truth
Edge/IoT botnet malware is often memory-resident — little on disk, no "scan
button," and a reboot may wipe it. Detection leans on **behavioral signs** and
**network observation**, not antivirus.

### Routers & IoT (primary targets)
Behavioral red flags:
- Device running hot, sluggish, rebooting on its own
- High outbound traffic when idle
- DNS settings changed unexpectedly (classic compromise tell)
- Admin password no longer works (attacker changed it)
- Unknown port-forwarding or firewall rules
- Unknown devices in the connected-clients list

Check in the admin page: DNS servers, port forwarding, firewall/remote-management
settings, firmware version. Confirm **remote/WAN management is OFF**. Check vendor
site for a known CVE matching your version.

### Computers (Win/macOS/Linux)
- Run a reputable anti-malware scan (Defender, Malwarebytes; macOS: KnockKnock).
- Inspect outbound connections for unknown hosts:
  - Windows: `netstat -ano` + Task Manager, or Resource Monitor
  - macOS/Linux: `lsof -i`, `netstat -an`; Little Snitch / GlassWire
- Check startup items, scheduled tasks, cron jobs.
- (Pure recon botnets like JDY mostly target edge devices, not laptops.)

### From the outside
- Search your **public IP** on shodan.io to see what the internet can see.
- Watch for ISP/vendor botnet-notification services.

## Hardening checklist (the real fix)

- [ ] **Patch firmware promptly** on routers/cameras/IoT — single biggest defense
- [ ] **Disable remote/WAN admin access** on everything
- [ ] **Replace end-of-life devices** that no longer get security updates
- [ ] **Change default credentials**; strong, unique admin passwords
- [ ] **Segment IoT** onto a separate VLAN / guest Wi-Fi
- [ ] Periodically check your public IP on Shodan

### Quick triage if a device looks compromised
1. **Reboot** — wipes memory-resident malware.
2. **Update firmware immediately** — reboot alone leaves you vulnerable; JDY
   rescans constantly and reinfects within minutes.
3. If still suspicious: **factory reset → update firmware before reconnecting →
   set a strong new admin password.**
