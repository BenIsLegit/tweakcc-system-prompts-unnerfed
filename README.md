# tweakcc system prompts — un-nerfed edition

A public, downloadable set of modified [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) system prompts. Every `.md` file under [`system-prompts/`](./system-prompts/) is a drop-in replacement for one of the prompts that Claude Code ships to the model — re-written to remove the "be as brief and minimal as possible" directives that the stock prompts pile on, and to replace them with instructions that ask for thorough, senior-engineer-grade work.

These are the actual files I use daily on my own machine. This repository is the publicly shared mirror of `~/.tweakcc/system-prompts/` — the working directory that [tweakcc](https://github.com/Piebald-AI/tweakcc) extracts Claude Code's system prompts into so I can edit them. Nothing is reconstructed or cleaned up for public consumption; this is the live set, including all of my in-progress un-nerfs.

**Currently aligned with: Claude Code v2.1.116.** Upstream [tweakcc](https://github.com/Piebald-AI/tweakcc) has not yet been updated for Claude Code releases newer than v2.1.113 — until it catches up, use the [**`tweakcc-fixed`**](https://github.com/BenIsLegit/tweakcc-fixed) fork (`npx tweakcc-fixed@latest`), which bundles the required upstream PRs plus several additional fixes. See [Which tweakcc fork to use](#which-tweakcc-fork-to-use-while-upstream-catches-up) below for details.

When Anthropic ships a newer Claude Code release, the supported re-patch flow is: **clear** `~/.tweakcc/system-prompts/`, **re-run tweakcc** to regenerate fresh stock `.md` files, **copy them into this repo's `system-prompts/`** (overwriting), **run [`python scripts/apply-unnerfs.py`](./scripts/apply-unnerfs.py)** to re-apply every un-nerf idempotently, **resolve any `FAIL`s** against drifted upstream wording (normal git-reviewable edits here), then **copy the un-nerfed result back to `~/.tweakcc/system-prompts/`** and run tweakcc's apply step. This wipe-then-regen path is required because tweakcc does **not** overwrite user-edited `.md` files — on conflict it writes an HTML diff file next to each changed prompt and leaves the `.md` alone, which would otherwise leave your working directory in a half-upgraded state. Full workflow documented in [Maintenance](#maintenance-re-applying-un-nerfs-after-a-claude-code-version-bump).

---

## Table of contents

- [What this repo is](#what-this-repo-is)
- [Background: what is tweakcc?](#background-what-is-tweakcc)
- [The two sources of changes](#the-two-sources-of-changes)
  - [Source 1: `roman01la`'s patch-claude-code.sh gist](#source-1-roman01las-patch-claude-codesh-gist)
  - [Source 2: my local `system-prompts` repo](#source-2-my-local-system-prompts-repo)
- [Import process, step by step](#import-process-step-by-step)
- [The un-nerf thesis](#the-un-nerf-thesis)
- [Concrete before/after examples](#concrete-beforeafter-examples)
- [Repository layout](#repository-layout)
- [File categories](#file-categories)
- [Compatibility notes](#compatibility-notes)
- [How to use these prompts yourself](#how-to-use-these-prompts-yourself)
- [Maintenance: re-applying un-nerfs after a Claude Code version bump](#maintenance-re-applying-un-nerfs-after-a-claude-code-version-bump)
- [Credits](#credits)
- [License / disclaimer](#license--disclaimer)

---

## What this repo is

- **272 markdown files** that Claude Code loads as system prompts, agent prompts, skill bodies, tool descriptions, and reference data blobs. The count tracks upstream — when Anthropic adds or removes prompts in a Claude Code release, this number changes to match.
- Each file has YAML-in-HTML-comment frontmatter giving it a human-readable name, a one-line description, and the Claude Code version the prompt was extracted from (e.g. `ccVersion: 2.1.116`).
- The body of each file is the literal prompt text that Claude Code assembles and sends to the model.
- Every file on disk is either:
  - a **stock extraction** from tweakcc (unchanged upstream text, at whatever ccVersion the prompt was last touched upstream); or
  - a **modified version** where I have edited the prompt to flip a "be brief / be minimal / be concise" directive into "be thorough / be complete / use the space the work warrants."
- **A re-apply script** at [`scripts/apply-unnerfs.py`](./scripts/apply-unnerfs.py) restores every un-nerf idempotently. It reads the current state of `system-prompts/`, runs each rule against the matching file, and prints a structured per-file report showing what applied, what was already un-nerfed (skipped), what failed (with a diff-friendly explanation), and which files had their line endings normalized. Running the script is the supported way to bring a fresh tweakcc extraction (e.g. after an upstream bump) back to the un-nerfed state.

This is a *snapshot* of a working directory, not a curated patch set. The README, the script's rule definitions, and the file contents together are the documentation of what was changed and why.

---

## Background: what is tweakcc?

[tweakcc](https://github.com/Piebald-AI/tweakcc) is a community tool that makes Claude Code's system prompts editable. Claude Code ships as a compiled Bun-packaged native binary (since v2.1.113); its system prompts are baked into the binary as string literals.

When you install and run tweakcc, it:

1. Locates the Claude Code binary on disk.
2. Extracts every system prompt, tool description, agent prompt, skill body, and reference data blob into a tree of `.md` files under `~/.tweakcc/system-prompts/` (on Windows: `C:\Users\<you>\.tweakcc\system-prompts\`).
3. Records a hash of each original prompt in `systemPromptOriginalHashes.json`.
4. On demand, recompiles the binary, substituting your edited `.md` contents back into the native binary, and records the applied hashes in `systemPromptAppliedHashes.json`.
5. Keeps a backup of the original binary (`native-binary.backup`, ~234 MB) so you can always roll back.

Result: you can edit any Claude Code system prompt with a normal text editor, save, run tweakcc, and have the running Claude Code CLI use your edits from that point forward. That folder — `~/.tweakcc/system-prompts/` — is exactly what this repository mirrors.

### Why tweakcc, and not the gist's patcher?

The script in [roman01la's gist](https://gist.github.com/roman01la/483d1db15043018096ac3babf5688881) (more on it below) predates the native-binary era of Claude Code. It worked by installing Claude Code from npm (which shipped plain JavaScript `cli.js`), running a series of `sed`-style string replacements against the prompt text, and repointing the `claude` symlink to the patched npm build. As of Claude Code **v2.1.113** Anthropic moved to a compiled Bun native binary with bytecode integrity checks — the gist's approach is obsolete against the current release.

tweakcc is the modern equivalent: same philosophy (edit the prompts), different mechanism (binary patching with hash verification and rollback). This repo's files are the tweakcc-format equivalents of the patches the gist was originally trying to apply, extended to cover many more prompts than the gist touched.

### Which tweakcc fork to use (while upstream catches up)

Upstream [Piebald-AI/tweakcc](https://github.com/Piebald-AI/tweakcc) (last published release `4.0.11`, based on commit `2e1d03e` — _Prompts for 2.1.113_) has **not** yet been updated for Claude Code releases newer than v2.1.113. Running stock upstream tweakcc against a v2.1.114+ binary fails at apply time because several patch regexes don't match the new minified shapes, and a few long-open upstream PRs that are required for the tool to function on recent CC builds haven't merged yet.

Until upstream catches up, use the [**BenIsLegit/tweakcc-fixed**](https://github.com/BenIsLegit/tweakcc-fixed) fork, published to npm as [`tweakcc-fixed`](https://www.npmjs.com/package/tweakcc-fixed). It bundles the required upstream PRs (#601 WASMagic import guard, #646 React Compiler output support, #655 Bun bytecode fallback + `clearBytecode`, #664 `\"` handling) plus additional fixes on top (scoped backslash-doubling, `verbose:X` destructure guard, adapted 2.1.113 minified-shape regexes, and a pile of `userMessageDisplay` fixes for theme bg, padding, wrapped-line bg, and the long-paste `[object Object]` bug). The fork targets Claude Code up to and including **2.1.116**.

Install via npx — always with `@latest` because the fork iterates frequently as new CC versions ship:

```bash
npx tweakcc-fixed@latest            # interactive UI
npx tweakcc-fixed@latest --apply    # apply customizations from ~/.tweakcc/config.json
```

Everywhere this README says "run tweakcc", substitute `npx tweakcc-fixed@latest` for now. Once the required PRs land upstream, switch back to plain `tweakcc` — the `~/.tweakcc/` layout and config format are identical between the two.

---

## The two sources of changes

The edits in this repo come from two distinct inputs, layered in that order:

### Source 1: `roman01la`'s patch-claude-code.sh gist

**URL:** https://gist.github.com/roman01la/483d1db15043018096ac3babf5688881

**What it is:** A bash script that applies 11 targeted string replacements to Claude Code's `cli.js`. Each replacement flips a "minimum viable / be brief / don't gold-plate" instruction into a "correct and complete / senior-engineer-grade / add validation at real boundaries" instruction.

**The gist's thesis:** Claude Code's stock system prompts contain roughly 15–20 separate directives telling the model to be brief, minimal, or concise, versus only 3–4 directives telling it to be thorough. That ~5:1 ratio biases the model toward corner-cutting. The gist tries to rebalance by selectively strengthening the "thorough" side and weakening the "brief" side — **only for implementation decisions**, not for chat brevity (it deliberately leaves the "keep chat messages short" directives alone because those don't make Claude write worse code).

**The gist's 11 patches, in plain English:**

| # | Target directive | Direction of change |
|---|---|---|
| 1 | "simplest approach that works" | → "correctly and completely solves" |
| 2 | "do the minimum the task requires" | → "do the work a senior developer would do" |
| 3 | "don't add abstractions beyond what the task requires" | softened to allow reasonable cleanup |
| 4 | Anti-gold-plating clause | carve-out: may fix obviously broken adjacent code related to the task |
| 5 | Error-handling guidance | explicitly add validation at real boundaries (I/O, network, user input, external APIs) |
| 6 | "don't refactor" | softened |
| 7 | "match the scope of the request" | → "address closely related issues you discover when fixing them is clearly the right thing to do" |
| 8 | Explore-agent "as quickly as possible" | removed; completeness preferred |
| 9 | Final-report brevity cap on subagents | removed |
| 10 | "one-line docstrings max" | removed; meaningful docs allowed |
| 11 | "2-sentence end-of-turn summary" cap | removed; scale to the work |

**A/B evidence from the gist author:** Porting Box2D (~30k lines of C) to JavaScript. Unpatched Claude Code produced 1,419 lines with an O(n²) broad phase and no sub-stepping. Patched Claude Code produced 1,885 lines (+33%) with a dynamic AABB tree, 4-level sub-stepping, and soft contact constraints. The patched run produced an actual physics engine port; the unpatched run produced a toy.

**How these changes landed in this repo:** I did **not** run the script. The script targets a file format (`cli.js`) and a distribution channel (npm) that no longer applies to current Claude Code. Instead, the patches served as the *template* for my initial hand-translated edits. The first round of changes — the gist-inspired "init patches" — is a direct translation of the gist's targeted edits into the tweakcc `.md` format. Specifically:

- `system-prompt-tone-concise-output-short.md` — flipped "short and concise" to "clear and appropriately detailed" (later strengthened further).
- `system-prompt-doing-tasks-no-unnecessary-error-handling.md` — flipped "don't add error handling for scenarios that can't happen" to "add validation at real boundaries where failures can realistically occur."
- `system-prompt-executing-actions-with-care.md` — added the carve-out "address closely related issues you discover when fixing them is clearly the right thing to do."
- `system-prompt-agent-thread-notes.md` — flipped "include code snippets only when exact text is load-bearing" to "include code snippets when they provide useful context."
- `agent-prompt-explore.md` — removed "as quickly as possible" directive, replaced with "Be thorough in your exploration."
- `agent-prompt-general-purpose.md` — flipped "don't gold-plate but don't leave it half-done" to "Do the work that a careful senior developer would do, including edge cases and fixing obviously related issues you discover."

Those six files are the gist's ideas, ported to tweakcc-compatible markdown.

### Source 2: my local `system-prompts` repo

**Location on my machine:** `~/.tweakcc/system-prompts/` (the tweakcc working directory, which has its own local git history).

**What it is:** The live tweakcc working directory, with a git log covering every edit I've made on top of a clean v2.1.113 stock extraction. The history starts from the v2.1.113 stock baseline, layers on the gist-inspired init patches described above, and then iterates through a series of un-nerf commits that extend the gist's thesis to many more prompts than the original script touched.

**Why a local git repo at all:** Because tweakcc writes directly into this directory and running it (without care) could overwrite edits, I keep it under version control so a bad re-extraction or a merge with a newer ccVersion is reversible. The commit history also documents *which* prompts were changed, *why*, and in *what order* — information that would otherwise be lost.

**What the local repo adds on top of the gist:** a series of un-nerf changes extending the gist's flip to many additional prompts. In the rough order they were applied:

- Mid-turn updates can use the space they need — no one-sentence hammer.
- End-of-turn summaries scale with the work — no hard 2-sentence cap; real summary with enough depth that the user can understand what happened without re-reading the diff.
- Meaningful code comments and docstrings allowed — no "one line max." Well-commented code is a feature, not bloat.
- Unleashed the core communication-style and thinking-frequency prompts to favor depth, rationale, and extended reasoning over token minimization. The "penalty for overthinking" framing is removed.
- Subagent and explore prompts now demand thorough reports and exhaustive exploration; final reports include file paths, code excerpts, reasoning, edge cases, and related observations instead of a minimal "what was done" blurb.
- Tool-usage, compaction, loop-check, and thread-notes prompts demand thorough reports and full-context summaries.
- PR-review, dream/memory summaries, learning-insights, batch-recipe, and memory notes demand thorough output.
- Loop/cron schedule confirmations and team-onboarding walkthroughs give full context and reasoning instead of "briefly confirm" one-liners.
- SendUserMessage and ultraplan prompts ask for substantive, detailed output instead of "tight" one-liners.
- Auto-mode rule reviewer and sandbox-restriction explainer produce thorough critiques and full restriction context.
- Unleashed subagent usage — removed caps like "use the minimum number of subagents," "not excessively," and "do not spawn subagents unless clearly necessary." Liberal parallel subagent use is explicitly encouraged.
- Final lint / whitespace pass.
- **Re-apply script added** ([`scripts/apply-unnerfs.py`](./scripts/apply-unnerfs.py)) so that tweakcc re-extracts against newer Claude Code binaries no longer require hand-reverting every un-nerfed file. The script stores each un-nerf as a `(stock_text, unnerf_text, description)` rule, applies them idempotently against the current working copy, and prints a structured report showing exactly which rules applied, which were already in place, and — critically — which ones failed to match because upstream text drifted. See the [Maintenance](#maintenance-re-applying-un-nerfs-after-a-claude-code-version-bump) section for the full workflow.

---

## Import process, step by step

This is how the contents of this GitHub repo were produced from the two sources above:

1. **tweakcc extraction.** I ran tweakcc against my installed Claude Code v2.1.113 binary. tweakcc wrote 271 `.md` files into `~/.tweakcc/system-prompts/`, one per extractable prompt/tool-description/skill/data-blob. This was the stock baseline.
2. **Gist-inspired initial patch.** Working from the gist's 11 string replacements, I hand-translated each into the equivalent edit on the tweakcc `.md` files. Six files changed.
3. **Iterative un-nerf edits.** Over the next set of edits I extended the thesis. Whenever I caught Claude Code being reflexively terse in a way that hurt the output (e.g., "here's the fix, no explanation"), I traced the behavior to the prompt that caused it, edited that prompt, and committed. That produced the stack of un-nerfs summarized above under [Source 2](#source-2-my-local-system-prompts-repo).
4. **Mirror copy to this public repo.** I copied the entire `.md` set from `~/.tweakcc/system-prompts/` into `~/.tweakcc/system-prompts-github/system-prompts/`, preserving the filenames and structure tweakcc uses. That is what you're browsing right now.
5. **Git init + this README.** Initialized `system-prompts-github` as a fresh git repo (the public mirror has its own history separate from the private working repo) and wrote this README to document the import.

The repo is then kept in sync with newer Claude Code releases via [`scripts/apply-unnerfs.py`](./scripts/apply-unnerfs.py); see [Maintenance](#maintenance-re-applying-un-nerfs-after-a-claude-code-version-bump) for the ongoing workflow.

**What this repo deliberately does NOT include:**

- `native-binary.backup` (234 MB — Claude Code's original binary; not mine to redistribute)
- `native-claudejs-orig.js` / `native-claudejs-patched.js` (12 MB each; same reason)
- `systemPromptOriginalHashes.json` / `systemPromptAppliedHashes.json` (specific to my machine's tweakcc run)
- `config.json` (tweakcc's local configuration)
- `prompt-data-cache/` (ephemeral)
- `.claude/`, `.serena/` (my local tool state)

If you want the hash files or the binary backup, run tweakcc yourself against your own Claude Code install; they'll be regenerated.

---

## The un-nerf thesis

Every edit in this repo is grounded in one observation:

> **Claude Code's stock prompts contain many more instructions to be brief, minimal, and concise than instructions to be thorough.**

Count them yourself — they cluster into four buckets:

1. **Chat-brevity directives** — "respond in 2-3 sentences," "match response length to the request," "terse one-liner is fine." These are about the *text Claude sends to the user*. They are mostly fine; nobody wants a wall of text for "what's the git status."
2. **Implementation-brevity directives** — "don't add abstractions," "don't gold-plate," "match the scope of the request," "simplest approach that works." These are about the *code Claude writes*. These are the ones the gist and this repo flip, because they cause Claude to produce correct-looking but shallow implementations.
3. **Process-brevity directives** — "as quickly as possible," "don't explore more than necessary," "report back concisely," "2-sentence summary." These are about the *work Claude does*. These cause Claude to under-investigate, miss edge cases, and underreport findings.
4. **Thoroughness directives** — "think step by step," "consider edge cases," "check your work." These exist but are outnumbered 5:1 by the previous three categories.

The un-nerf principle: **keep bucket 1, flip buckets 2 and 3, amplify bucket 4.**

That's it. Every edit in this repo fits that rule. I am not trying to make Claude verbose; I am trying to make Claude *thorough*. The two are not the same, and the stock prompts conflate them.

---

## Concrete before/after examples

These show the literal text change between the upstream stock prompt and the current un-nerfed version on disk.

### Example 1 — `system-prompt-tone-concise-output-short.md`

**Stock:**
> Your responses should be short and concise.

**Current:**
> Your responses should be thorough, clear, and rich with explanation, reasoning, and context. Favor depth and completeness over brevity — the user benefits from understanding the full picture, including tradeoffs, related observations, and the reasoning behind decisions. There is no word limit; use whatever length the task genuinely warrants to produce genuinely helpful output.

### Example 2 — `system-prompt-doing-tasks-no-unnecessary-error-handling.md`

**Stock:**
> Don't add error handling, fallbacks, or validation for scenarios that can't happen. Trust internal code and framework guarantees. Only validate at system boundaries (user input, external APIs). Don't use feature flags or backwards-compatibility shims when you can just change the code.

**Current:**
> Add error handling and validation at real boundaries where failures can realistically occur (user input, external APIs, I/O, network). Trust internal code and framework guarantees for truly internal paths. Don't use feature flags or backwards-compatibility shims when you can just change the code.

Subtle but important: the stock version leads with the prohibition ("don't add"). The patched version leads with the requirement ("add … at real boundaries"). Same safety caveat, opposite default.

### Example 3 — `agent-prompt-general-purpose.md` (subagent system prompt)

**Stock:**
> You are an agent for Claude Code … Complete the task fully—don't gold-plate, but don't leave it half-done. When you complete the task, respond with a concise report covering what was done and any key findings — the caller will relay this to the user, so it only needs the essentials.

**Current:**
> You are an agent for Claude Code … Complete the task fully and thoroughly. Do the work that a careful senior developer would do, including edge cases and fixing obviously related issues you discover. Don't add purely cosmetic or speculative improvements unrelated to the task. When you complete the task, respond with a thorough, detailed report covering what was done, every key finding, the reasoning behind decisions, edge cases you considered, and any related observations the caller should know about. The caller relies on your report to understand the full picture — do not minimize detail.

### Example 4 — `system-prompt-communication-style.md` (the big one)

**Stock (summarized):** "Briefly state what you're about to do. Useful updates at key moments. Don't narrate. Keep summaries short. Match response length to task size. Comments only where they genuinely help — avoid noise."

**Current (summarized):** "Explain what you're about to do *and why*. Substantive updates at every key moment including when you reason through a tradeoff. Walk the user through your reasoning when non-obvious. Full explanations beat cryptic one-liners. End-of-turn summary scales with the work — real summary with enough depth, not a token-minimizing stub. Never withhold useful context. Well-commented code is a feature, not bloat."

Full diff is too long to reproduce here — read the file itself under [`system-prompts/system-prompt-communication-style.md`](./system-prompts/system-prompt-communication-style.md).

### Example 5 — `system-reminder-thinking-frequency-tuning.md`

**Stock:** "Tune your thinking frequency — on simpler user messages, respond or act directly without thinking unless further reasoning is necessary. On more complex tasks, you should feel free to reason as much as needed for best results but without overthinking. Avoid unnecessary thinking in response to simple user messages."

**Current:** "Think as deeply and as often as the work benefits from — extended reasoning produces better results, catches edge cases, and surfaces issues that shallow responses miss. There is no penalty for thorough thinking; use it whenever careful reasoning would improve the answer, the plan, or the implementation. On complex tasks, think extensively; on simpler ones, think enough to verify your approach is actually correct before acting."

This is one of the highest-leverage un-nerfs. The system reminder that gets injected into every user message was actively telling Claude to *think less*; the patched version tells it to think as much as the work warrants and explicitly removes the "penalty for overthinking" framing.

---

## Repository layout

```
system-prompts-github/
├── README.md                    <- this file
├── .git/                        <- public-mirror git history
└── system-prompts/              <- 272 markdown files, mirror of ~/.tweakcc/system-prompts/
    ├── agent-auto-mode-rule-reviewer.md
    ├── agent-prompt-*.md        <- subagent / auto-agent system prompts (37 files)
    ├── data-*.md                <- reference data blobs: API refs, model catalog, etc. (33 files)
    ├── skill-*.md               <- user-facing skill bodies (27 files)
    ├── system-prompt-*.md       <- core system prompts (most of the 98 in this bucket)
    ├── system-reminder-*.md     <- system-reminder templates injected into user messages
    ├── tool-description-*.md    <- descriptions shown to the model for each built-in tool (77 files)
    └── tool-parameter-*.md      <- parameter-level descriptions for tool inputs
```

File counts are approximate; the full inventory is whatever `ls system-prompts/` shows you.

---

## File categories

Counted by filename prefix:

| Prefix | Count | What these are |
|---|---|---|
| `system-prompt-*` / `system-reminder-*` | **98** | The main set — core behavioral instructions, tone, task-execution guidance, system reminders injected into messages. Most of the un-nerfs target files in this bucket. |
| `tool-description-*` / `tool-parameter-*` | **77** | The `description` and parameter-level copy shown to the model for each built-in tool (Read, Write, Edit, Bash, Grep, Glob, Agent, WebFetch, WebSearch, TaskCreate, etc.). These shape how the model decides *when* to use which tool. Mostly left stock; a few tweaks around tool-usage prompts. |
| `agent-prompt-*` / `agent-auto-*` | **37** | Full subagent system prompts — explore, general-purpose, plan, code-reviewer, security-review, onboarding, dream-memory, and many more. Heavy un-nerf territory because subagents are where over-brevity is worst (the caller can't see into a subagent's thinking, so if it under-reports the user never knows). |
| `data-*` | **33** | Reference data embedded in prompts — Anthropic API reference (per language), model catalog, HTTP error codes, live documentation sources, managed-agents docs. Mostly stock — these are facts, not behavior. |
| `skill-*` | **27** | User-facing skill bodies (e.g., `skill-simplify.md`, `skill-debugging.md`, `skill-init-claude-md-and-skill-setup-new-version.md`). Mixed — some un-nerfed, some stock. |

**Total: 272 files.** Every file is ≤ 33 KB, plain markdown. You can open and read any of them without any tooling.

---

## Compatibility notes

- **Claude Code version.** These files are aligned with Claude Code v2.1.116. Individual prompts carry their own `ccVersion:` frontmatter ranging from v2.0.14 (oldest surviving prompt) to the current release. When Anthropic ships a newer Claude Code, prompts may be added, removed, or re-worded upstream. Getting a clean re-extract requires **clearing** `~/.tweakcc/system-prompts/` first — tweakcc does not overwrite user-edited files, it writes an HTML diff alongside each conflicted prompt, which is why the maintenance workflow wipes the directory before re-running tweakcc. After the clean re-extract, copy the fresh stock into this repo and run [`python scripts/apply-unnerfs.py`](./scripts/apply-unnerfs.py) here to idempotently re-apply every un-nerf; the script prints a per-file report telling you exactly which rules need updating if upstream text has drifted. `systemPromptOriginalHashes.json` is how tweakcc knows which prompts are unchanged vs. modified on a given binary; the script doesn't use those hashes directly — it works purely on stock-text-in-file detection.
- **Model family.** These prompts are tuned for current Claude models (Opus 4.7 / Sonnet 4.6 / Haiku 4.5 as of January 2026). Older models may follow the un-nerfed prompts differently — in particular, the "think more, verbose is fine, use space the work warrants" directives may cause older/smaller models to over-explain even simple responses. Test on your own workload.
- **Risk of over-verbosity.** This is the main failure mode to watch for. If you apply all of these and suddenly Claude Code is giving you a 15-paragraph essay in response to "what time is it?", that's because the un-nerfed communication prompt is instructing it to be thorough. The [un-nerf thesis](#the-un-nerf-thesis) tried to preserve chat brevity for simple requests, but there's always going to be some spillover. If you see this, the first place to look is `system-prompt-communication-style.md` and `system-prompt-tone-concise-output-short.md`.
- **Token cost.** Thorough output uses more tokens than brief output. Plan accordingly.

---

## How to use these prompts yourself

**Option A — with tweakcc-fixed (recommended):**

Use the [**`tweakcc-fixed`**](https://github.com/BenIsLegit/tweakcc-fixed) fork for now — see [Which tweakcc fork to use](#which-tweakcc-fork-to-use-while-upstream-catches-up) for why. No global install required; `npx` fetches a fresh copy each run.

1. Clone this repo.
2. **If `~/.tweakcc/system-prompts/` already exists and contains anything** (from a prior tweakcc run, or any hand-edits), clear it first:
   - Unix: `rm -rf ~/.tweakcc/system-prompts`
   - Windows PowerShell: `Remove-Item -Recurse -Force "$HOME\.tweakcc\system-prompts"`

   Leave the rest of `~/.tweakcc/` alone — state files (`config.json`, `systemPromptOriginalHashes.json`, `native-binary.backup`) must survive. This step matters because tweakcc does **not** overwrite user-edited `.md` files on re-extract — it writes an HTML diff file next to each conflicted prompt and leaves the original `.md` in place. Wiping the directory first guarantees a clean fresh-stock extraction with no diff HTMLs.
3. Run `npx tweakcc-fixed@latest` once against your Claude Code install. It extracts a fresh stock copy of every prompt into `~/.tweakcc/system-prompts/`.
4. Copy the `.md` files from this repo's `system-prompts/` directory over the ones in `~/.tweakcc/system-prompts/`, overwriting. **Do NOT overwrite `~/.tweakcc/systemPromptOriginalHashes.json` or any other tweakcc state files** — only the `.md` files.
5. Run `npx tweakcc-fixed@latest --apply` (or the interactive "Apply customizations" action). It will detect the changed hashes, re-patch the Claude Code binary, and record the new applied hashes.
6. Restart any running Claude Code sessions.

Example Unix-ish copy (adapt paths for your OS):

```
git clone <this-repo-url> ~/src/tweakcc-system-prompts-github
rm -rf ~/.tweakcc/system-prompts
npx tweakcc-fixed@latest                                                           # fresh stock extract
cp -r ~/src/tweakcc-system-prompts-github/system-prompts/*.md ~/.tweakcc/system-prompts/
npx tweakcc-fixed@latest --apply                                                   # patch the CC binary
```

**Option B — read-only reference:**

You don't have to apply these to use this repo. If you're building your own prompt engineering pipeline (e.g., a custom agent with the Claude API / Anthropic SDK), read the files here to see what Claude Code ships versus what I think it should ship. The diffs between ccVersion comments and un-nerfed body text are effectively a prompt-engineering case study on brevity-vs-thoroughness tradeoffs.

**Option C — cherry-pick:**

Most prompts stand alone. If you only want the un-nerfed [`system-prompt-communication-style.md`](./system-prompts/system-prompt-communication-style.md) and nothing else, just copy that one file. Each file's frontmatter tells you what it governs, and the body is self-contained.

---

## Maintenance: re-applying un-nerfs after a Claude Code version bump

Every Claude Code release can rewrite, add, remove, or subtly shift the wording of any system prompt. Because tweakcc does **not** overwrite user-edited `.md` files on re-extract (it writes an HTML diff alongside each conflicted file and leaves the original alone), the maintenance workflow uses this repo as the git-tracked source of truth: wipe `~/.tweakcc/system-prompts/` for a clean stock extract, copy that into this repo, replay every un-nerf against the fresh stock with:

```
python scripts/apply-unnerfs.py
```

then copy the result back. Full step-by-step below under [Standard workflow after a Claude Code bump](#standard-workflow-after-a-claude-code-bump).

The script is stdlib-only (no pip install), idempotent, and safe to run repeatedly. It also normalizes any accidental CRLF line endings back to LF.

### What the script does

- Reads every `.md` file under `./system-prompts/` (override with `--dir PATH`).
- For each file listed in its `RULES` table, applies a series of string replacements of the form `stock_text → unnerf_text`. Each rule is typically one paragraph's worth of un-nerf, so a single file may have multiple rules.
- For every rule, one of four things happens:
  - **APPLIED** — the stock text was found in the file and replaced with un-nerfed text. The file is rewritten to disk as UTF-8 LF.
  - **SKIPPED** — the un-nerfed text is already in the file (nothing to do; the run is idempotent).
  - **NORMALIZED** — no rule content change was needed, but the file had CRLF line endings and was rewritten as LF.
  - **FAIL** — neither stock nor un-nerfed text was found. This means upstream drifted — the expected passage has been reworded by Anthropic. The report quotes the first 200 characters of both the expected stock text and the intended un-nerf text so you can search the file and update the rule.
- Prints a per-file report and a summary block with counts + exit code.

### Useful flags

| Flag | What it does |
|---|---|
| (none) | Apply all rules to `./system-prompts/`. Writes files on disk. |
| `--dry-run` | Report what would change without writing any files. |
| `--check` | Like `--dry-run`, but exits with code 1 if **any** rule would apply or fail. Useful in CI/pre-commit to assert the working copy is already un-nerfed. |
| `--only FILE` | Restrict processing to a single filename (no path — just e.g. `system-prompt-communication-style.md`). Handy when iterating on one rule. |
| `--verbose` | Include context on SKIPPED entries too (otherwise they're shown but not explained). |
| `--dir PATH` | Process a prompts directory other than `./system-prompts/`. |

### Standard workflow after a Claude Code bump

> **Why the dance with `~/.tweakcc/system-prompts/` instead of just re-running tweakcc in place:** tweakcc does **not** overwrite user-edited `.md` files on re-extract. When it detects that a prompt has changed upstream but the local `.md` has been hand-edited — which is true for every un-nerfed file in this repo — it writes an HTML diff file alongside the `.md` and leaves the original untouched. That's a reasonable default for a tool with no separate source-of-truth, but since **this repo is the git-tracked source of truth** for the un-nerfed content, the cleanest flow is: wipe the tweakcc working directory so it has no user edits to protect, let tweakcc emit a fresh clean stock extract, copy that stock into this repo (overwriting the tree here — `git diff` then shows exactly what Anthropic changed upstream), run the re-apply script **here** against the fresh stock, resolve any `FAIL`s via normal git-reviewable edits, commit, and finally copy the fully un-nerfed tree back to `~/.tweakcc/system-prompts/` for tweakcc to compile into the binary.

> Substitute `npx tweakcc-fixed@latest` for `tweakcc` below while upstream catches up — see [Which tweakcc fork to use](#which-tweakcc-fork-to-use-while-upstream-catches-up).

1. **Update Claude Code** the normal way (installer, npm upgrade, whatever).
2. **Clear `~/.tweakcc/system-prompts/`.**
   - Unix: `rm -rf ~/.tweakcc/system-prompts`
   - Windows PowerShell: `Remove-Item -Recurse -Force "$HOME\.tweakcc\system-prompts"`

   Leave the rest of `~/.tweakcc/` alone — `config.json`, `systemPromptOriginalHashes.json`, `systemPromptAppliedHashes.json`, and `native-binary.backup` must survive. Wiping just the `system-prompts/` subdirectory is what lets tweakcc do a conflict-free re-extract.
3. **Re-extract with tweakcc.** `npx tweakcc-fixed@latest` (or stock `tweakcc` once upstream is caught up). tweakcc sees the missing `system-prompts/` directory, reads the new CC binary, and writes fresh stock `.md` files — no diff HTMLs emitted, because there are no user edits to conflict with.
4. **Copy the fresh stock into this repo, overwriting.** From the repo root: `cp ~/.tweakcc/system-prompts/*.md system-prompts/`. Do *not* commit yet — `git diff` at this point shows exactly which prompts moved upstream, by how much, and whether any un-nerfed passages got re-worded. That's useful context for the next step.
5. **Run the re-apply script.** `python scripts/apply-unnerfs.py`. Read the report.
6. **Address any `FAIL` entries:**
   - Open the named file, locate the passage the rule targets (use the quoted stock text from the report as a search term — you may need to search for a partial phrase if drift is significant).
   - Compare the new stock wording against the old stock wording and against the un-nerf. Usually the drift is cosmetic (a reworded sentence) and the un-nerf still applies after updating `stock` in the rule to match the new wording byte-exactly.
   - In rare cases upstream may have removed the passage entirely or changed it structurally enough that the un-nerf no longer applies. Delete the rule in that case and note the removal in the commit message.
   - Re-run the script until it prints only APPLIED / SKIPPED / NORMALIZED — no FAILs.
7. **Scan for *new* prompts** Anthropic added in this release. They show up as untracked `.md` files in `git status`. For each new prompt, decide whether it introduces a brevity nerf that fits this repo's thesis and add a rule if so. Some new prompts (e.g. structured JSON generators with UX-driven word caps) should be left stock; use the bucket taxonomy from [The un-nerf thesis](#the-un-nerf-thesis) to decide.
8. **Commit** the updated `.md` files and the updated `scripts/apply-unnerfs.py` rules together, while the diff is still small and the context is fresh.
9. **Copy the un-nerfed tree back out** to `~/.tweakcc/system-prompts/`. From the repo root: `cp system-prompts/*.md ~/.tweakcc/system-prompts/`. This is the copy tweakcc will actually bake into the patched binary.
10. **Run tweakcc's apply step** so the patched binary picks up the new `.md` content: `npx tweakcc-fixed@latest --apply` (or the interactive "Apply customizations" action).
11. **Restart any running Claude Code sessions** and verify a representative un-nerfed prompt (e.g. the communication-style one) actually made it into the running binary by triggering the relevant behavior.

### Adding a new un-nerf

1. Identify the passage you want to flip. Read the relevant `.md` file.
2. Open [`scripts/apply-unnerfs.py`](./scripts/apply-unnerfs.py) and add a new `Rule(stock=..., unnerf=..., description=...)` entry under the matching filename key in the `RULES` dict (create the key if the file is brand new).
   - `stock` must be exactly what's in the file right now (the un-nerfed-ish default). Byte-exact, including any trailing whitespace or unusual punctuation — template literals like `${VAR}` and backticks go in verbatim.
   - `unnerf` is the replacement. Write it in the same thorough-over-brief voice as the rest of the repo.
   - `description` is a short scannable label (e.g. `"tone body: flip 'short and concise' to 'thorough, clear, rich'"`).
3. Run `python scripts/apply-unnerfs.py --dry-run --only <filename>` to preview.
4. Run without `--dry-run` once you're happy with the preview.
5. Verify with `git diff` that the only change to the file is the replacement you expected.
6. Run `python scripts/apply-unnerfs.py --check` to confirm the rule is fully idempotent.
7. Commit the script change and the un-nerfed `.md` together.

### When a rule fails (understanding the report)

A `FAIL` entry in the report looks like:

```
system-prompts/some-file.md
  [FAIL    ] <rule description>
             Expected stock text (first 200 chars):
               'Briefly explain what ...'
             Expected un-nerf text (first 200 chars, for reference):
               'Explain thoroughly what ...'
             Neither was found in the file.
             Action: open C:\...\some-file.md and locate the passage the rule targets.
             If upstream text drifted, update the rule's `stock` field in
             scripts/apply-unnerfs.py to match the new upstream wording.
```

This format is deliberately structured so that Claude (or a future-you) reading the report can act on it without re-deriving what the rule was supposed to do. The two quoted excerpts tell you both what you're looking for (the pre-drift stock) and what the result should be after un-nerfing (the target unnerf); the file path tells you exactly where to go.

The three most common causes of a FAIL:

- **Upstream drifted the stock wording.** Anthropic reworded the passage in a new Claude Code release. Open the file, find the new wording of the same passage, and update `stock` in the rule to match it byte-exactly. The `unnerf` usually doesn't need changing unless the new upstream phrasing is structurally different.
- **Upstream removed the passage.** The nerfed directive got deleted from the prompt entirely. Delete the rule (it's no longer relevant) and note this in your commit message.
- **Upstream replaced the passage with something that was never nerfed in the first place** (e.g., the brevity directive got replaced with a neutral or pro-thoroughness one). Delete the rule — the un-nerf isn't needed anymore.

### CI / pre-commit integration (optional)

Because `--check` exits 1 when anything would change, you can wire the script into a pre-commit hook to assert the working copy is fully un-nerfed before each commit:

```
python scripts/apply-unnerfs.py --check || {
  echo "Un-nerfs are not fully applied. Run: python scripts/apply-unnerfs.py"
  exit 1
}
```

---

## Credits

- **[tweakcc](https://github.com/Piebald-AI/tweakcc)** by Piebald AI — the tool that made any of this possible. Without tweakcc I would be hex-editing a Bun binary.
- **[roman01la's patch-claude-code.sh gist](https://gist.github.com/roman01la/483d1db15043018096ac3babf5688881)** — the original thesis (15:3 brevity-vs-thoroughness imbalance) and the first 11 patches, which I translated into the initial tweakcc-format edits in this repo.
  - **PR's #8 textual refinement** — widened the phrase gate in [`agent-prompt-explore.md`](./system-prompts/agent-prompt-explore.md) from the literal `"very thorough"` to any caller request for `thorough` exploration, so the exhaustive-search clause now fires on a broader range of caller phrasings. Applied here and in the upstream tweakcc working copy.
- **Anthropic** — for Claude Code, and for not going out of their way to stop community patching.

---

## License / disclaimer

The system prompt *content* in `system-prompts/*.md` was extracted from Claude Code by tweakcc and then modified by me. The prompt text itself is Anthropic's copyright (it's part of a commercial product). I am redistributing a modified subset of it under fair-use / research-use terms, on the same understanding the tweakcc project operates under.

The README, documentation, and organization in this repo are released under **CC0 / public domain** — use freely, no attribution required, no warranty implied.

**This is not an Anthropic-endorsed, Anthropic-supported, or Anthropic-sanctioned set of prompts.** Applying these will change the behavior of Claude Code in ways that may be unwanted or counter-productive for your workload. Test in a throwaway session first. Keep the tweakcc binary backup so you can roll back. Your mileage will vary.

If you find a prompt here that behaves badly, open an issue (or a PR) — the un-nerf process is ongoing and I'm always interested in cases where "thorough" tipped over into "obnoxious."
