# Godot Contribution — Phases 0 & 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Get the human onboarded to Godot Engine's codebase and contribution process, with a working local build of the engine and enough orientation in the networking + crypto focus area to pick a first issue.

**Architecture:** Two-phase ramp. Phase 0 is read-only: docs, source-tree survey, community lurking, PR study. Phase 1 is a working local toolchain: prerequisites installed, repo cloned, first full build succeeds, incremental edit→rebuild loop verified against a file in the focus area.

**Tech Stack:** Godot Engine (C++17), SCons 4.0+ build system, Xcode Command Line Tools (clang, make, git), Python 3, MoltenVK / Vulkan SDK. Host: macOS on Apple Silicon (arm64, verified via `uname -m`).

**Artifacts created in this repo:** this plan file, plus up to two lean notes files appended as learning accumulates:
- `source-tree-map.md` — one-line notes per focus-area path (Task 2)
- `build-setup.md` — working build command + any macOS quirks hit (Task 6)

No logs, PR trackers, or weekly journals. Append learnings to the two notes files only when worth keeping.

**Focus area reminder:** networking + crypto / TLS. The paths that matter:
- `core/crypto/`, `core/io/`
- `modules/mbedtls/`, `modules/enet/`, `modules/websocket/`, `modules/webrtc/`

---

## Phase 0 — Orient (week 0–1, ~4–6 hrs total)

### Task 1: Read Godot's contributor documentation

**Files:** None in this repo yet. You're reading the web.

- [ ] **Step 1: Read the contributing site's engine introduction**

Open and read top-to-bottom: <https://contributing.godotengine.org/en/latest/engine/introduction.html>

Expected time: 15–25 min. This page explains what "contributing to the engine" actually means and what the moving parts are.

- [ ] **Step 2: Read the PR workflow page**

Open and read: <https://contributing.godotengine.org/en/latest/pull_requests/pr_workflow.html>

Pay attention to: branch policy, commit message conventions, testing requirements, AI-disclosure norms.

- [ ] **Step 3: Read the review guidelines**

Open and read: <https://contributing.godotengine.org/en/latest/pull_requests/review_guidelines.html> and <https://contributing.godotengine.org/en/latest/organization/pull_requests/merge_guidelines.html>

Key numbers to internalize: **80% of regression fixes merged/rejected in 1 week; 80% of bug fixes in 2 weeks.** If your PR is silent past those, you're expected to escalate in Contributors' chat (that's the documented norm, not rudeness).

- [ ] **Step 4: Skim the best-practices doc**

Open and skim (don't memorize): <https://contributing.godotengine.org/en/latest/engine/guidelines/best_practices.html>

Don't commit anything yet — this task is read-only.

### Task 2: Map the focus-area source tree

**Files:**
- Create: `godot-contributions/source-tree-map.md`

- [ ] **Step 1: Create the source-tree-map file**

Create `/Users/oargaruna/learning-notes/godot-contributions/source-tree-map.md` with this skeleton:

```markdown
# Godot source-tree map (networking + crypto focus)

One-liner per file/subfolder. Skim, don't read deeply.

## core/crypto/

## core/io/ (networking-relevant files only)

## modules/mbedtls/

## modules/enet/

## modules/websocket/

## modules/webrtc/
```

- [ ] **Step 2: Skim `core/crypto/` via GitHub web UI**

Open <https://github.com/godotengine/godot/tree/master/core/crypto> in a browser.

For each `.h` and `.cpp` file in that folder: read the top comment block and the class declarations in the `.h`. Add a one-liner under `## core/crypto/` in your notes file. Don't read implementations.

Expected output (example shape, not exact):
```markdown
## core/crypto/
- `crypto.h` — `Crypto`, `CryptoKey`, `X509Certificate`, `HashingContext` public APIs
- `crypto.cpp` — stub/dispatch to active provider (mbedTLS)
- `crypto_core.h/cpp` — primitives (RandomNumberGenerator, hashing contexts)
- `aes_context.h/cpp` — AES modes exposed to scripts
- `hashing_context.h/cpp` — incremental hashing (SHA-1/256/512)
```

- [ ] **Step 3: Skim `core/io/` for networking-relevant files only**

Open <https://github.com/godotengine/godot/tree/master/core/io>.

Only map files whose names start with `stream_peer`, `tcp_server`, `packet_peer`, or `http_client`. Skip everything else in this directory — `resource_format_*`, `file_access_*`, `image_loader_*` and similar are outside the focus area.

- [ ] **Step 4: Skim each of the four focus-area modules**

For each of these, open the GitHub tree view and write one-liners:
- <https://github.com/godotengine/godot/tree/master/modules/mbedtls>
- <https://github.com/godotengine/godot/tree/master/modules/enet>
- <https://github.com/godotengine/godot/tree/master/modules/websocket>
- <https://github.com/godotengine/godot/tree/master/modules/webrtc>

Time-box each to ~5 min. Goal is "I know where X lives," not "I understand X."

- [ ] **Step 5: Commit the source-tree map**

```bash
git -C /Users/oargaruna/learning-notes add godot-contributions/source-tree-map.md
git -C /Users/oargaruna/learning-notes commit -m "godot: map focus-area source tree"
```

Expected: one commit, one file added.

### Task 3: Join Contributors' chat and lurk

**Files:** None.

- [ ] **Step 1: Join the chat**

Open <https://chat.godotengine.org/>. Sign up (works with GitHub auth or email).

- [ ] **Step 2: Join the key channels**

Join these channels (search by name): `contributors`, `networking`, and `build-system`. If a networking channel doesn't exist under that name, use the closest match — don't spend more than 2 min searching.

- [ ] **Step 3: Lurk, don't post**

Scroll back ~1 week in each channel. Read. Do not post yet. Calibrate: how formal is the tone, how long are messages, how quickly do core contributors reply, what kinds of questions are welcome.

Expected time: 20–30 min. No artifact produced. Just absorb.

### Task 4: Read 3–5 recently merged PRs in the focus area

**Files:**
- Optionally append to: `godot-contributions/source-tree-map.md` under a new `## PR reading notes` section. Only if you captured something worth remembering.

- [ ] **Step 1: Find candidate PRs**

Open this search URL in a browser:

<https://github.com/godotengine/godot/pulls?q=is%3Apr+is%3Amerged+label%3Atopic%3Anetwork+sort%3Aupdated-desc>

Also try:

<https://github.com/godotengine/godot/pulls?q=is%3Apr+is%3Amerged+crypto+in%3Atitle>

Pick 3–5 merged PRs from the last ~6 months. Prefer small/medium ones (< ~500 lines changed) over giant refactors.

- [ ] **Step 2: For each PR, read end-to-end**

For each selected PR, read in order:
1. The PR description
2. The linked issue (if any) — for context
3. All review comments, top to bottom
4. The final commit(s) — especially how the code changed in response to review

Time per PR: 15–25 min.

- [ ] **Step 3: Write down what surprised you (optional)**

If anything surprised you — a convention you didn't expect, a quality bar you noticed, a maintainer comment worth quoting — add it to `source-tree-map.md` under a `## PR reading notes` section. Skip if nothing stuck.

- [ ] **Step 4: Commit if notes were added**

```bash
git -C /Users/oargaruna/learning-notes status godot-contributions/
git -C /Users/oargaruna/learning-notes add godot-contributions/source-tree-map.md
git -C /Users/oargaruna/learning-notes commit -m "godot: notes from reading focus-area PRs"
```

Expected: either a commit with a diff to `source-tree-map.md`, or `git status` shows no changes and you skip the commit.

**Phase 0 milestone:** you've read the contributor docs, mapped the focus-area tree, joined chat, and seen what merged PRs actually look like. You should now be able to read an open issue in the focus area and form an opinion about scope.

---

## Phase 1 — Local build working (week 1–2, ~4–8 hrs)

### Task 5: Install macOS build prerequisites

**Files:** None in this repo.

- [ ] **Step 1: Verify / install Xcode Command Line Tools**

Run:
```bash
xcode-select -p
```

Expected output: a path like `/Library/Developer/CommandLineTools` or `/Applications/Xcode.app/Contents/Developer`.

If the command prints `xcode-select: error: unable to get active developer directory...`, install the tools:
```bash
xcode-select --install
```
Accept the GUI prompt. Installation takes 10–20 min.

- [ ] **Step 2: Verify Python 3 and pip**

```bash
python3 --version
python3 -m pip --version
```

Expected: Python 3.10+ and a pip version. macOS Sonoma+ ships Python 3.9+; any 3.9+ is fine for SCons.

- [ ] **Step 3: Install SCons 4.0+**

Prefer Homebrew if you use it:
```bash
brew install scons
```

Or via pip (user install, avoids system Python conflicts):
```bash
python3 -m pip install --user scons
```

Verify:
```bash
scons --version
```
Expected: `SCons: v4.x.y` — anything 4.0 or newer.

- [ ] **Step 4: Install Vulkan SDK (for MoltenVK)**

macOS doesn't ship Vulkan. Godot 4.x needs MoltenVK for the rendering backend at runtime.

Open <https://vulkan.lunarg.com/sdk/home#mac> and download the latest macOS SDK installer (`.dmg`). Run it. Accept defaults — the installer sets up the SDK under `~/VulkanSDK/<version>/` and writes environment setup scripts.

Source the SDK env for your current shell (add to `~/.zshrc` if you want it persistent):
```bash
source ~/VulkanSDK/*/setup-env.sh
```

Verify:
```bash
echo $VULKAN_SDK
```
Expected: a non-empty path like `/Users/oargaruna/VulkanSDK/1.3.x.y/macOS`.

- [ ] **Step 5: Sanity check the toolchain**

```bash
which clang && clang --version
which make && make --version | head -1
which git && git --version
```

Expected: all three resolve to paths, all print versions. No "command not found."

### Task 6: Clone the repo and run the first full build

**Files:**
- Create: `godot-contributions/build-setup.md`
- The Godot source lives outside this repo — put it under `~/src/godot` (or your preferred code directory). It is **not** a submodule of learning-notes.

- [ ] **Step 1: Choose a location and clone**

```bash
mkdir -p ~/src
git clone --depth=1 https://github.com/godotengine/godot.git ~/src/godot
```

Expected: clone finishes in 30–90 seconds. `--depth=1` trims history for speed; you can later `git fetch --unshallow` if you need the full log.

- [ ] **Step 2: Create the build-setup notes file**

Create `/Users/oargaruna/learning-notes/godot-contributions/build-setup.md`:

```markdown
# Godot build setup (macOS arm64)

Host: macOS, Apple Silicon (arm64).
Godot checkout: `~/src/godot` (shallow clone from master).

## Working build command

```
cd ~/src/godot
scons platform=macos target=editor arch=arm64 dev_build=yes compiledb=yes scu_build=yes
```

Time: _TBD — fill in after first build_.
Output binary: _TBD — fill in after first build, e.g. `bin/godot.macos.editor.dev.arm64`_.

## Flags explained

- `platform=macos` — build for macOS
- `target=editor` — build the full editor (vs. `template_debug`/`template_release` for runtime)
- `arch=arm64` — Apple Silicon
- `dev_build=yes` — debug symbols, faster rebuilds, runtime asserts on — correct for contributors
- `compiledb=yes` — generate `compile_commands.json` for clangd / IDE integration
- `scu_build=yes` — single-compilation-unit build; dramatically faster full builds

## macOS quirks hit

_(append any errors + resolutions here as they happen)_
```

(This file starts with placeholders you'll fill in after the first build. That's intentional — it becomes a record of what actually worked.)

- [ ] **Step 3: Run the first full build**

```bash
cd ~/src/godot
scons platform=macos target=editor arch=arm64 dev_build=yes compiledb=yes scu_build=yes
```

Expected: 10–25 min on modern Apple Silicon. Output lands in `bin/`. The binary name will be something like `godot.macos.editor.dev.arm64` (the exact suffix depends on flags and Godot version — check the `bin/` directory).

If the build fails, do **not** rerun blindly. Read the error. Common macOS failure modes:
- Missing Vulkan SDK env (re-source `setup-env.sh`)
- Xcode CLT not accepted (`sudo xcodebuild -license accept`)
- Wrong arch (Rosetta / x86 Python mismatch) — verify `python3 -c "import platform; print(platform.machine())"` prints `arm64`

Log the error and resolution under `## macOS quirks hit` in `build-setup.md`.

- [ ] **Step 4: Fill in the build-setup notes**

Replace the two TBDs in `build-setup.md` with real values:

```bash
ls ~/src/godot/bin/
```

Copy the actual editor binary name into the "Output binary" line. Note the wall-clock build time.

- [ ] **Step 5: Launch the editor**

```bash
~/src/godot/bin/godot.macos.editor.dev.arm64
```

(Substitute the actual binary name from Step 4.)

Expected: Godot's project manager window opens. You don't need to create a project — just confirm the editor launches without crashing.

Close it.

- [ ] **Step 6: Commit the build-setup notes**

```bash
git -C /Users/oargaruna/learning-notes add godot-contributions/build-setup.md
git -C /Users/oargaruna/learning-notes commit -m "godot: record working macOS build command"
```

### Task 7: Verify the incremental edit → rebuild → run loop

The entire rest of your contribution effort depends on this cycle being fast and reliable. Prove it now.

**Files:**
- Modify (temporarily, in the Godot clone): `~/src/godot/core/crypto/crypto.cpp`
- Optionally append to: `godot-contributions/build-setup.md`

- [ ] **Step 1: Make a trivial, reversible edit in the focus area**

Open `~/src/godot/core/crypto/crypto.cpp` in your editor. Find any function — the `Crypto::generate_random_bytes` method is a reasonable target.

Add a `print_line` at the top of the function body:

```cpp
print_line("godot-contrib: crypto touched");
```

`print_line` is Godot's standard console-print function and is already in scope via the crypto module's includes. If it isn't, add `#include "core/string/print_string.h"` at the top of the file.

- [ ] **Step 2: Rebuild incrementally**

```bash
cd ~/src/godot
scons platform=macos target=editor arch=arm64 dev_build=yes compiledb=yes scu_build=yes
```

(Same command as first full build. SCons handles incremental automatically.)

Expected: **much** faster than the full build. A single-file change should rebuild in under ~60 seconds on Apple Silicon.

If SCU-build means one file change triggers a large SCU unit recompile, that's normal. If it takes over 3 min for a single-line edit, something is wrong — note it in `build-setup.md` under `## macOS quirks hit`.

- [ ] **Step 3: Run the editor and confirm the change is live**

```bash
~/src/godot/bin/godot.macos.editor.dev.arm64
```

In the project manager, open any project (create a blank one if needed). Trigger anything that exercises `Crypto::generate_random_bytes` — easiest way: open the script editor, add a one-liner node script like:

```gdscript
extends Node
func _ready():
    var c := Crypto.new()
    print(c.generate_random_bytes(4))
```

Run the scene.

Expected: the editor's output panel prints `godot-contrib: crypto touched` followed by the random bytes. That confirms your edit is running.

- [ ] **Step 4: Revert the edit**

```bash
cd ~/src/godot
git diff core/crypto/crypto.cpp  # inspect what you changed
git checkout core/crypto/crypto.cpp
```

Expected: diff is clean after checkout.

- [ ] **Step 5: Record the feedback-loop time**

Open `godot-contributions/build-setup.md` and append at the bottom:

```markdown
## Incremental build timing

- Single-file edit in `core/crypto/crypto.cpp` → rebuild time: _<fill in>_
- Editor launch time: _<fill in>_
- End-to-end edit→see-change loop: _<fill in>_
```

Fill in the real numbers. This is the number that determines whether the next 6 months are pleasant or painful.

- [ ] **Step 6: Commit the timing notes**

```bash
git -C /Users/oargaruna/learning-notes add godot-contributions/build-setup.md
git -C /Users/oargaruna/learning-notes commit -m "godot: record incremental build timing"
```

**Phase 1 milestone:** working local Godot build on macOS arm64, compile_commands.json generated for IDE support, incremental edit→rebuild→run loop verified against a file in the focus area.

---

## Phase 0 + 1 exit criteria

All of these true before moving on to Phase 2 (first PR):

- [ ] Read the four core contributor docs (engine intro, PR workflow, review guidelines, merge guidelines).
- [ ] `godot-contributions/source-tree-map.md` committed with one-liners for the six focus-area paths.
- [ ] Joined Godot Contributors' chat, lurked in `#contributors` and at least one networking-adjacent channel.
- [ ] Read 3–5 merged PRs in the focus area end-to-end.
- [ ] Xcode CLT, Python 3, SCons 4.0+, Vulkan SDK all verified installed.
- [ ] `~/src/godot` cloned; full build succeeds.
- [ ] Editor launches from `bin/godot.macos.editor.dev.arm64` (or equivalent).
- [ ] `godot-contributions/build-setup.md` committed with real build command, timing, and any macOS quirks hit.
- [ ] Incremental edit in `core/crypto/` rebuilds and runs; loop timing recorded.

If any of these is blocked for more than ~1 hr of troubleshooting, capture the blocker in `build-setup.md` under `## macOS quirks hit` and ask in Godot Contributors' chat. That's the documented escalation path — use it.
