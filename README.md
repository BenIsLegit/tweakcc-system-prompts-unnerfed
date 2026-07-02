> [!IMPORTANT]
> **This repo is unmaintained.** I've switched to Codex and don't really run Claude Code anymore, so I can't tell whether prompt edits for new releases actually help. And I'm not going to ship un-nerfs I haven't tested. The files here still work as a reference, and they still apply against the Claude Code version listed below. But don't wait around for updates when a new release lands.
>
> If you want something that's kept up to date, use [lukehutch/unnerfcc](https://github.com/lukehutch/unnerfcc). It started as a fork of this repo and has drifted a long way since. It comes with an installer script that handles the whole setup, plus instructions that let Claude un-nerf new releases on its own, so it keeps pace with new Claude Code versions. This one doesn't.

# tweakcc system prompts — un-nerfed edition

Modified [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) system prompts that remove the "be brief, be minimal" directives and replace them with instructions to be thorough. These are the actual files I use daily. Nothing here is cleaned up for public consumption — this is the live set, including all in-progress un-nerfs.

> [!NOTE]
> Currently aligned with **Claude Code v2.1.142**.

|  |  |
|---|---|
| [Install](#install) | Get these running on your machine |
| [The thesis](#the-un-nerf-thesis) | Why these edits exist |
| [Examples](#beforeafter-examples) | What the changes look like |
| [Maintenance](MAINTENANCE.md) | Keeping up with new Claude Code releases |
| [Background](BACKGROUND.md) | How tweakcc works, where these edits came from |

---

## Install

You need [tweakcc](https://github.com/Piebald-AI/tweakcc) to patch these into your Claude Code binary.

Grab the un-nerfed prompts as a release zip — no clone needed. From [Releases](https://github.com/BenIsLegit/tweakcc-system-prompts-unnerfed/releases), pick the tag whose Claude Code version matches yours (e.g. `v2.1.142-1` if you're on Claude Code v2.1.142) and download the `system-prompts-v<version>-<iteration>.zip` asset.

```bash
# 1. Wipe any existing tweakcc prompts (tweakcc won't overwrite edited files,
#    so a clean slate avoids conflicts)
rm -rf ~/.tweakcc/system-prompts          # Unix
# Remove-Item -Recurse -Force "$HOME\.tweakcc\system-prompts"  # Windows

# 2. Extract fresh stock prompts from your Claude Code binary
npx tweakcc@latest

# 3. Drop the un-nerfed prompts on top of the stock ones (overwriting stock).
#    Replace <PATH-TO-DOWNLOADED-ZIP> with where you saved the release asset.
unzip -o <PATH-TO-DOWNLOADED-ZIP> -d ~/.tweakcc/system-prompts/   # Unix
# Expand-Archive -Force <PATH-TO-DOWNLOADED-ZIP> "$HOME\.tweakcc\system-prompts" # Windows

# 4. Patch the binary
npx tweakcc@latest --apply

# 5. Restart any running Claude Code sessions
```

Leave the rest of `~/.tweakcc/` alone when you wipe `system-prompts/`. Files like `config.json`, `systemPromptOriginalHashes.json`, and `native-binary.backup` need to survive.

You don't have to patch anything to get value from this repo. The files work as a prompt-engineering reference on their own — browse [`system-prompts/`](system-prompts) on GitHub, or unzip the release asset and read them locally. The diffs between stock and un-nerfed text are a case study in how brevity directives shape model behavior. And if you only want one or two changes, cherry-pick individual `.md` files. Each one is self-contained.

---

## The un-nerf thesis

Claude Code's stock prompts contain way more instructions to be brief than instructions to be thorough. Count them yourself. They fall into four groups:

1. **Chat brevity** — "respond in 2-3 sentences," "terse one-liner is fine." These control the text Claude sends you. Mostly fine. Nobody wants a wall of text for "what's the git status."
2. **Implementation brevity** — "don't add abstractions," "don't gold-plate," "simplest approach." These control the code Claude writes. They cause shallow implementations.
3. **Process brevity** — "as quickly as possible," "report back concisely," "2-sentence summary." These control how Claude investigates and reports. They cause under-investigation and under-reporting.
4. **Thoroughness** — "think step by step," "consider edge cases." These exist, but they're outnumbered roughly 5:1.

The principle: **keep group 1, flip groups 2 and 3, amplify group 4.**

That's it. Every edit in this repo follows that rule. The goal isn't making Claude verbose. It's making Claude thorough. The stock prompts treat those as the same thing, and they're not.

---

## Before/after examples

### Tone directive (`system-prompt-tone-concise-output-short.md`)

**Stock:**
> Your responses should be short and concise.

**Un-nerfed:**
> Your responses should be thorough, clear, and rich with explanation, reasoning, and context. Favor depth and completeness over brevity — the user benefits from understanding the full picture, including tradeoffs, related observations, and the reasoning behind decisions. There is no word limit; use whatever length the task genuinely warrants to produce genuinely helpful output.

### Error handling (`system-prompt-doing-tasks-no-unnecessary-error-handling.md`)

**Stock:**
> Don't add error handling, fallbacks, or validation for scenarios that can't happen. Trust internal code and framework guarantees. Only validate at system boundaries (user input, external APIs). Don't use feature flags or backwards-compatibility shims when you can just change the code.

**Un-nerfed:**
> Add error handling and validation at real boundaries where failures can realistically occur (user input, external APIs, I/O, network). Trust internal code and framework guarantees for truly internal paths. Don't use feature flags or backwards-compatibility shims when you can just change the code.

The stock version leads with the prohibition ("don't add"). The un-nerfed version leads with the requirement ("add ... at real boundaries"). Same safety caveat, opposite default.

### Thinking frequency (`system-reminder-thinking-frequency-tuning.md`)

**Stock:** "Tune your thinking frequency — on simpler user messages, respond or act directly without thinking unless further reasoning is necessary. [...] Avoid unnecessary thinking."

**Un-nerfed:** "Think as deeply and as often as the work benefits from — extended reasoning produces better results, catches edge cases, and surfaces issues that shallow responses miss. There is no penalty for thorough thinking."

This is one of the highest-leverage changes. The stock prompt was actively telling Claude to think less. The un-nerfed version removes the "penalty for overthinking" framing entirely.

---

## Repo layout

```
system-prompts-github/
├── README.md
├── MAINTENANCE.md
├── BACKGROUND.md
├── scripts/
│   └── apply-unnerfs.py          # re-applies all un-nerfs after a CC version bump
└── system-prompts/               # ~294 markdown files
    ├── system-prompt-*.md        # core behavioral instructions, tone, task guidance (~65)
    ├── system-reminder-*.md      # injected into user messages (~40)
    ├── tool-description-*.md     # tool descriptions shown to the model (~81)
    ├── tool-parameter-*.md       # parameter-level tool descriptions
    ├── agent-prompt-*.md         # subagent system prompts (~37)
    ├── data-*.md                 # reference data blobs (~40)
    └── skill-*.md                # user-facing skill bodies (~30)
```

Counts are approximate. The full inventory is whatever `ls system-prompts/` shows.

---

## Compatibility

- **Claude Code version:** Aligned with v2.1.142. Individual prompts carry `ccVersion:` frontmatter ranging from v2.0.14 to the current release. When Anthropic ships a new version, see [MAINTENANCE.md](MAINTENANCE.md) for the update workflow.
- **Model family:** Tuned for current Claude models (Opus 4.7 / Sonnet 4.6 / Haiku 4.5). Older or smaller models might over-explain simple responses with these prompts active.
- **Over-verbosity:** This is the main failure mode to watch for. If Claude starts writing essays in response to "what time is it?", look at `system-prompt-communication-style.md` and `system-prompt-tone-concise-output-short.md` first.
- **Token cost:** Thorough output uses more tokens. Plan accordingly.

---

## Credits

- **[tweakcc](https://github.com/Piebald-AI/tweakcc)** by Piebald AI — the tool that makes all of this possible.
- **[roman01la's patch-claude-code.sh gist](https://gist.github.com/roman01la/483d1db15043018096ac3babf5688881)** — the original thesis and first 11 patches, which I translated into tweakcc format. See [BACKGROUND.md](BACKGROUND.md#the-original-gist) for the full breakdown.
- **Anthropic** — for Claude Code, and for not going out of their way to stop community patching.

---

## License / disclaimer

The prompt text in `system-prompts/*.md` was extracted from Claude Code by tweakcc, then modified. The original prompt content is Anthropic's copyright. I'm redistributing a modified subset under fair-use / research-use terms, same basis the tweakcc project operates on.

The original code (the scripts in `scripts/`) is licensed **MIT** — see [`LICENSE`](LICENSE).

The README, docs, and repo organization are **CC0 / public domain**.

**This is not Anthropic-endorsed or Anthropic-supported.** Applying these will change Claude Code's behavior in ways that might not suit your workflow. Test in a throwaway session first. Keep the tweakcc binary backup so you can roll back.

If something here behaves badly, open an issue or PR.
