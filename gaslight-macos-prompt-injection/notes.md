# Gaslight — macOS malware that prompt-injects the analyst's AI tools

Notes from The Hacker News, 2026-06.
Source: https://thehackernews.com/2026/06/new-gaslight-macos-malware-uses-prompt.html

## Who

Previously undocumented Rust-based macOS implant + infostealer, assessed
**with high confidence** to be the work of **North Korea–aligned threat actors**.
Conventional DPRK-style credential/espionage harvesting — but with a novel
anti-analysis twist (below).

## The headline technique: prompt injection aimed at the *defender*

The payload is targeted not at the victim, but at the **malware analyst's
LLM-assisted triage pipeline**. Reverse engineers increasingly paste strings /
decompiled code into AI tools to summarize and analyze samples; Gaslight tries to
**poison that step**.

Buried in the sample is a Markdown-fenced block containing **38 fabricated
"system messages"** — fake errors about token expiry, out-of-memory kills, disk
exhaustion, repeated operation failures, plus bogus "injection vulnerability"
warnings. If an analyst feeds the sample to an LLM, the model may ingest these as
real context/instructions, get derailed, and produce a misleading or aborted
analysis.

The name fits: the malware tries to make the analyst's tooling **doubt its own
reality**. This is the genuinely new development — attackers treating defenders'
AI tools as an attack surface.

**Takeaway:** treat any "system messages" / log strings found *inside* a binary
as attacker-controlled content, never ground truth. Don't pipe untrusted samples
straight into an LLM without isolation/sanitization.

## C2 over Telegram

Uses the **Telegram Bot API** as its command channel — enters a polling loop so
the operator gets an interactive shell. Commands:

- `help`
- `id` — identify the implant
- `shell` — execute commands via `execvp`
- `kill` — terminate a process by PID
- `upload` — exfiltrate a file via Telegram
- `stop` — halt execution
- (suspected 7th `focus` command, unexplained)

Runtime hygiene: bot token / chat ID / operator settings are supplied **at
runtime, not hard-coded**, and the implant **self-redacts its own bot token** in
runtime output to frustrate analysis.

## Infection chain & persistence

- **Persistence** via a LaunchAgent masquerading with an Apple-looking label:
  `com.apple.system.services.activity`.
- A small (~2 KB) **bash installer** drops a bundled `cpython-3.10.18`
  interpreter and runs an embedded **Base64-encoded Python stealer**.

## What it steals (Python component)

- Terminal command history, installed apps, running processes, hardware/software
  profile
- **macOS Keychain** databases
- **Browser data** — Chrome, Brave, Firefox, Safari

Zips everything into `temp/collected_data.zip` and exfiltrates over Telegram.

## Caveats / to verify

Sourced via a page summarizer — confirm exact strings against the original
article / underlying vendor report before citing: the LaunchAgent label, the
`cpython-3.10.18` version, the "38 system messages" count, and the suspected
`focus` command. Article gave **no explicit IOCs or formal mitigations**.
Dated 2026-06, past my training cutoff — attribution and technical claims are the
report's, not independently corroborated.
