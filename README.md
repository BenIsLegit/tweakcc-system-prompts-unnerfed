# tweakcc system prompts — un-nerfed edition

A public, downloadable set of modified [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) system prompts. Every `.md` file under [`system-prompts/`](./system-prompts/) is a drop-in replacement for one of the prompts that Claude Code ships to the model — re-written to remove the "be as brief and minimal as possible" directives that the stock prompts pile on, and to replace them with instructions that ask for thorough, senior-engineer-grade work.

These are the actual files I use daily on my own machine. This repository is the publicly shared mirror of `~/.tweakcc/system-prompts/` — the working directory that [tweakcc](https://github.com/Piebald-AI/tweakcc) extracts Claude Code's system prompts into so I can edit them. Nothing is reconstructed or cleaned up for public consumption; this is the live set, including all of my in-progress un-nerfs.

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
- [Credits](#credits)
- [License / disclaimer](#license--disclaimer)

---

## What this repo is

- **271 markdown files** that Claude Code loads as system prompts, agent prompts, skill bodies, tool descriptions, and reference data blobs.
- Each file has YAML-in-HTML-comment frontmatter giving it a human-readable name, a one-line description, and the Claude Code version the prompt was extracted from (e.g. `ccVersion: 2.1.113`).
- The body of each file is the literal prompt text that Claude Code assembles and sends to the model.
- Every file on disk is either:
  - a **stock v2.1.113 extraction** from tweakcc, unchanged; or
  - a **modified version** where I have edited the prompt to flip a "be brief / be minimal / be concise" directive into "be thorough / be complete / use the space the work warrants."

This is a *snapshot* of a working directory, not a curated patch set. The README and the file contents together are the documentation of what was changed and why.

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

---

## Import process, step by step

This is how the contents of this GitHub repo were produced from the two sources above:

1. **tweakcc extraction.** I ran tweakcc against my installed Claude Code v2.1.113 binary. tweakcc wrote 271 `.md` files into `~/.tweakcc/system-prompts/`, one per extractable prompt/tool-description/skill/data-blob. This was the stock baseline.
2. **Gist-inspired initial patch.** Working from the gist's 11 string replacements, I hand-translated each into the equivalent edit on the tweakcc `.md` files. Six files changed.
3. **Iterative un-nerf edits.** Over the next set of edits I extended the thesis. Whenever I caught Claude Code being reflexively terse in a way that hurt the output (e.g., "here's the fix, no explanation"), I traced the behavior to the prompt that caused it, edited that prompt, and committed. That produced the stack of un-nerfs summarized above under [Source 2](#source-2-my-local-system-prompts-repo).
4. **Mirror copy to this public repo.** I copied the entire `.md` set from `~/.tweakcc/system-prompts/` into `~/.tweakcc/system-prompts-github/system-prompts/`, preserving the filenames and structure tweakcc uses. That is what you're browsing right now.
5. **Git init + this README.** Initialized `system-prompts-github` as a fresh git repo (the public mirror has its own history separate from the private working repo) and wrote this README to document the import.

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

These show the literal text change between the stock v2.1.113 prompt and the current un-nerfed version on disk.

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
└── system-prompts/              <- 271 markdown files, mirror of ~/.tweakcc/system-prompts/
    ├── agent-auto-mode-rule-reviewer.md
    ├── agent-prompt-*.md        <- subagent / auto-agent system prompts (37 files)
    ├── data-*.md                <- reference data blobs: API refs, model catalog, etc. (33 files)
    ├── skill-*.md               <- user-facing skill bodies (27 files)
    ├── system-prompt-*.md       <- core system prompts (most of the 97 in this bucket)
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
| `system-prompt-*` / `system-reminder-*` | **97** | The main set — core behavioral instructions, tone, task-execution guidance, system reminders injected into messages. Most of the un-nerfs target files in this bucket. |
| `tool-description-*` / `tool-parameter-*` | **77** | The `description` and parameter-level copy shown to the model for each built-in tool (Read, Write, Edit, Bash, Grep, Glob, Agent, WebFetch, WebSearch, TaskCreate, etc.). These shape how the model decides *when* to use which tool. Mostly left stock; a few tweaks around tool-usage prompts. |
| `agent-prompt-*` / `agent-auto-*` | **37** | Full subagent system prompts — explore, general-purpose, plan, code-reviewer, security-review, onboarding, dream-memory, and many more. Heavy un-nerf territory because subagents are where over-brevity is worst (the caller can't see into a subagent's thinking, so if it under-reports the user never knows). |
| `data-*` | **33** | Reference data embedded in prompts — Anthropic API reference (per language), model catalog, HTTP error codes, live documentation sources, managed-agents docs. Mostly stock — these are facts, not behavior. |
| `skill-*` | **27** | User-facing skill bodies (e.g., `skill-simplify.md`, `skill-debugging.md`, `skill-init-claude-md-and-skill-setup-new-version.md`). Mixed — some un-nerfed, some stock. |

**Total: 271 files.** Every file is ≤ 33 KB, plain markdown. You can open and read any of them without any tooling.

---

## Compatibility notes

- **Claude Code version.** These files were extracted from Claude Code v2.1.113. Many of the prompts carry their own `ccVersion:` frontmatter; some are as old as v2.1.53, some as new as v2.1.107. When Anthropic ships a new Claude Code version, prompts may be added, removed, or re-worded upstream. Re-running tweakcc against the new version will re-extract fresh stock copies and you'll have to re-apply (or revisit) the un-nerfs. `systemPromptOriginalHashes.json` is how tweakcc knows which prompts are unchanged vs. modified.
- **Model family.** These prompts are tuned for current Claude models (Opus 4.7 / Sonnet 4.6 / Haiku 4.5 as of January 2026). Older models may follow the un-nerfed prompts differently — in particular, the "think more, verbose is fine, use space the work warrants" directives may cause older/smaller models to over-explain even simple responses. Test on your own workload.
- **Risk of over-verbosity.** This is the main failure mode to watch for. If you apply all of these and suddenly Claude Code is giving you a 15-paragraph essay in response to "what time is it?", that's because the un-nerfed communication prompt is instructing it to be thorough. The [un-nerf thesis](#the-un-nerf-thesis) tried to preserve chat brevity for simple requests, but there's always going to be some spillover. If you see this, the first place to look is `system-prompt-communication-style.md` and `system-prompt-tone-concise-output-short.md`.
- **Token cost.** Thorough output uses more tokens than brief output. Plan accordingly.

---

## How to use these prompts yourself

**Option A — with tweakcc (recommended):**

1. Install [tweakcc](https://github.com/Piebald-AI/tweakcc).
2. Run tweakcc once against your Claude Code install. This creates `~/.tweakcc/system-prompts/` with stock extractions.
3. Clone this repo.
4. Copy the `.md` files from this repo's `system-prompts/` directory over the ones in `~/.tweakcc/system-prompts/`, overwriting. **Do NOT overwrite `~/.tweakcc/systemPromptOriginalHashes.json` or any other tweakcc state files** — only overwrite the `.md` files.
5. Run tweakcc's apply command. It will detect the changed hashes, re-patch the Claude Code binary, and record the new applied hashes.
6. Restart any running Claude Code sessions.

Example Unix-ish copy (adapt paths for your OS):

```
git clone <this-repo-url> ~/src/tweakcc-system-prompts-github
cp -r ~/src/tweakcc-system-prompts-github/system-prompts/*.md ~/.tweakcc/system-prompts/
# then run tweakcc's apply action
```

**Option B — read-only reference:**

You don't have to apply these to use this repo. If you're building your own prompt engineering pipeline (e.g., a custom agent with the Claude API / Anthropic SDK), read the files here to see what Claude Code ships versus what I think it should ship. The diffs between ccVersion comments and un-nerfed body text are effectively a prompt-engineering case study on brevity-vs-thoroughness tradeoffs.

**Option C — cherry-pick:**

Most prompts stand alone. If you only want the un-nerfed [`system-prompt-communication-style.md`](./system-prompts/system-prompt-communication-style.md) and nothing else, just copy that one file. Each file's frontmatter tells you what it governs, and the body is self-contained.

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
