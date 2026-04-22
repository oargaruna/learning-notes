# Godot Engine contribution plan

Plan for becoming a recurring contributor to the Godot Engine open-source project, with contributions concentrated in the **networking + crypto / TLS** area where existing security/cloud background creates a skill edge.

## Why Godot, not Bevy

Bevy was the main alternative considered. Godot won on ramp-up cost given no prior Rust experience:

- Godot 4.x API is mature and stable; Bevy ships breaking API changes every ~3 months.
- Godot's contributor onboarding documentation is best-in-class for an AAA-scale OSS project (dedicated site at `contributing.godotengine.org`, engine-intro docs, explicit PR and review workflow); Bevy's user-facing docs are described as sparse.

- Godot has an official merge SLA (80% of bug fixes merged/rejected within 2 weeks); Bevy's process is active but less predictable.
- Bevy would require 4–8 weeks of Rust ramp-up before engine contributions could land. Godot's C++ is more transferable from a general systems background.

Bevy would be the better pick with prior Rust experience, or if learning Rust is itself part of the goal.

## Goal

Become a recurring contributor to Godot Engine with contributions concentrated in the networking + crypto lane, not scattered one-off PRs.

## Success criteria

1. **By +4 weeks:** local Godot build working; at least one trivial docs or typo PR in the focus area merged; `good first issue` triage list curated and a first code issue claimed.
2. **By +3 months:** at least one non-trivial code PR merged (real bug fix or small feature, not docs), active presence in the Contributors' chat, at least one PR review given on someone else's work.
3. **By +6 months:** recognized within the focus area — reviewers @-mentioning on related PRs or maintainers asking opinion on design questions.
4. **Ongoing cadence:** ~4–6 hours/week average, sustained.

## Non-goals

- Not shipping a game on Godot. This is a contributor path, not a user path.
- Not becoming a core maintainer in 6 months. That's a 2+ year arc and not the target.
- Not burning weekends on ramp-up that won't sustain. Steady weekly cadence beats sprint-burnout.

## Focus area: networking + crypto / TLS

Godot ships the following in-tree, all relevant:

- `core/crypto/` — `Crypto`, `CryptoKey`, `X509Certificate`, `HashingContext`, AES contexts
- `core/io/` — `StreamPeerTLS`, `StreamPeerTCP`, `HTTPClient`, `HTTPRequest`, packet/stream peer base classes
- `modules/mbedtls/` — mbedTLS integration (the crypto provider)
- `modules/enet/` — `ENetMultiplayerPeer`, UDP-based reliable networking
- `modules/websocket/` — WebSocket client/server
- `modules/webrtc/` — WebRTC data channels / peer connection

This is the area where TLS/PQC/identity background gives a real skill edge over the average Godot contributor. Issues in this area often sit unclaimed because few contributors are comfortable with the domain.

Realistic PR shapes:
- Fixing TLS handshake edge cases or error-reporting gaps
- Improving `Crypto` API ergonomics or filling missing cert-chain validation options
- WebRTC/WebSocket robustness fixes
- mbedTLS version bumps and integration cleanups

## Phased ramp

### Phase 0 — Orient (week 0–1, ~4–6 hrs total)

- Read `contributing.godotengine.org` end-to-end: engine intro, PR workflow, review guidelines, merge guidelines.
- Skim Godot source tree for the focus-area paths listed above. Map the territory, don't memorize.
- Join the Godot Contributors' chat. Lurk in networking-adjacent channels to calibrate tone and pace.
- Read 3–5 recently merged PRs in the focus area end-to-end (discussion + review + final diff). Highest-ROI activity in this phase.

### Phase 1 — Local build working (week 1–2, ~4–8 hrs)

- Clone `godotengine/godot`. Set up SCons + platform toolchain per official build docs.
- First full build (expect 20–25 min). Enable SCU/jumbo build and `compile_commands.json` for editor tooling.
- Confirm incremental build cycle: edit a file in `core/crypto/`, rebuild, run editor, see change. This feedback loop is what the next 6 months live in — fix it now if slow or flaky.

### Phase 2 — First merged PR (week 2–4, ~8–12 hrs)

- Scan `good first issue` plus networking/crypto-tagged issues.
- Start with a **docs or trivial fix PR in the focus area** — typo in `StreamPeerTLS` docs, unclear error message in `HTTPClient`, missing doc string on a `Crypto` method. Goal is to exercise the review loop, not prove technical chops yet.
- Submit following exact Godot conventions: commit message style, linked issue, AI-disclosure if applicable.
- **Milestone:** first PR merged by +4 weeks.

### Phase 3 — First non-trivial code PR (month 1–3, ~20–40 hrs)

- Pick one open networking/crypto issue with clear scope — bug with reproduction steps, or small feature already discussed in the issue thread. Confirm scope in the issue before writing code.
- Write fix + a test if the area has test coverage (crypto does; some areas don't).
- Work through review: expect 1–3 rounds of feedback. Be terse in responses; push back with evidence when warranted rather than rubber-stamping every suggestion.
- **Milestone:** non-trivial code PR merged by +3 months.

### Phase 4 — Focus-area recognition (month 3–6)

- Sustain ~1 PR/month cadence in the lane.
- **Review other people's PRs** in networking/crypto. Fastest path to "recognized" — maintainers notice helpful reviewers faster than one-more-contributor.
- Answer questions from less-experienced contributors when crypto/TLS topics come up in chat.
- Once credibility is established, propose one modest improvement (new `Crypto` API method, better cert-chain validation options, etc.). Iterate via issue discussion before writing code.

## Failure modes to pre-commit against

- **Speculative big PRs.** No "TLS 1.3 overhaul" without maintainer alignment first. Large unsolicited PRs are the top cause of stale PRs.
- **Stale-PR resignation.** If a PR has no reviewer after 2 weeks, escalate politely in Contributors' chat. Godot's docs explicitly encourage this.
- **Ramp burnout.** Under 2 hrs some week is fine. Missing several weeks in a row breaks the plan — re-plan rather than binge to catch up.
- **AI-disclosure.** Godot has explicit norms. If AI pair-coding is used, disclose it in the PR. Matters for community trust.

## Signals to review monthly

**On-track:**
- PRs merging
- Reviews arriving within Godot's ~2-week SLA
- Occasionally finishing a review of someone else's PR
- Contributors' chat starting to feel familiar

**Stalling — investigate:**
- No reviewer response after 2 weeks → escalate in chat
- Cadence below 2 hrs/week for 3+ weeks → re-plan, don't power through
- PRs keep getting fundamental redirection on approach → align earlier via issue discussion before coding

## Key references

- [Godot Contributing site](https://contributing.godotengine.org/) — primary reference
- [godotengine/godot repo](https://github.com/godotengine/godot)
- [`good first issue` label](https://github.com/godotengine/godot/labels/good%20first%20issue)
- [Godot Contributors' chat](https://chat.godotengine.org/) — join `#contributors` and networking-adjacent channels
- [godot-proposals repo](https://github.com/godotengine/godot-proposals) — feature discussions before code
- [Godot build docs](https://docs.godotengine.org/en/stable/contributing/development/compiling/index.html)

**Issue-search starters:**
- `is:issue is:open label:"topic:network"` in `godotengine/godot`
- `is:issue is:open crypto in:title` or `mbedtls in:title,body`
- `good first issue` label combined with networking terms for first-PR hunting
