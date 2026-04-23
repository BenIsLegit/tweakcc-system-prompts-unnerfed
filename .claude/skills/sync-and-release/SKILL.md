---
name: sync-and-release
description: End-to-end workflow to publish a new release after a Claude Code upstream version bump — re-applies un-nerfs to fresh stock, resolves any rule drift, commits the sync, tags `v<cc_version>-<iteration>`, and creates the GitHub release with the correct two-subsection body format. Use when fresh stock `.md` files from `~/.tweakcc/system-prompts/` have been copied into this repo (so `git status` shows the upstream delta as M/D/?) and the user wants to ship a new release.
---

# Sync prompts to a new Claude Code version and cut a release

This skill automates the post-upstream-bump release workflow documented in
[README.md § Standard workflow after a Claude Code bump](../../../README.md#standard-workflow-after-a-claude-code-bump).

## Preconditions (DO NOT bypass — check them before anything else)

1. **Working directory is the repo root.** Verify with `git remote -v` — it must point at
   `github.com/BenIsLegit/tweakcc-system-prompts-unnerfed`. If not, stop.
2. **Fresh stock has already been copied in.** `git status --short` must show a mix of
   `M` / `D` / `??` entries under `system-prompts/`. If the working tree is clean, the user
   hasn't done the wipe-and-regen step yet — tell them to follow README steps 1–4 first
   (`rm -rf ~/.tweakcc/system-prompts`, run `npx tweakcc-fixed@latest`, `cp ~/.tweakcc/system-prompts/*.md system-prompts/`).
3. **The user has authorized pushes and publishing.** This skill pushes commits, pushes
   tags, and publishes a GitHub release with a zip asset — all visible to others. If the
   user hasn't explicitly authorized these actions for this session, confirm once at the
   top before starting Phase 5.
4. **Do NOT run tweakcc itself.** The user runs tweakcc by hand against their local CC
   binary. This skill never touches `~/.tweakcc/` or the installed `claude` binary.

## Required reading

Before making any per-file judgment call, read:

- `README.md` — especially [§The un-nerf thesis](../../../README.md#the-un-nerf-thesis) (the
  bucket taxonomy). This is **load-bearing** for "should this new prompt get an un-nerf rule?"
  Without the bucket taxonomy you'll guess wrong.
- `scripts/apply-unnerfs.py` — top-of-file docstring plus the `RULES` dict. You may need to
  edit this file to update a drifted `stock` field or to add/remove a rule.

## Overall phases

1. **Inspect state & detect new ccVersion**
2. **Re-apply un-nerfs** (iterate until zero FAILs, zero missing)
3. **Bucket analysis on new / modified files** (decide whether each needs a new rule)
4. **Compose and commit the sync** (temp file + `-F`, surgical staging)
5. **Push the commit**
6. **Compute tag name, create annotated tag**
7. **Push the tag**
8. **Build the zip asset** (from the committed tree, not the working copy)
9. **Compose the release body** (two-subsection format)
10. **Publish the GitHub release**
11. **Verify** (Latest flag, asset, body render)
12. **Clean up temp files**

## Non-negotiable rules

These override default behavior — violating any of them breaks the repo's conventions:

- **NEVER** `git add -A` / `git add .` — stage files by name only.
- **NEVER** `git commit --amend` — use `git reset --soft HEAD~1` + re-commit.
- **NEVER** force-push to origin/master or to any shared tag.
- **NEVER** skip git hooks (`--no-verify`, `--no-gpg-sign`) — if a hook fails, fix the cause.
- **NEVER** include Claude (or yourself) as a commit co-author.
- **ALWAYS** write multi-paragraph commit / tag / release messages to a temp file and pass
  via `-F` or `--notes-file`. Passing them inline via `git commit -m "$(cat <<'EOF' ... EOF)"`
  strips blank lines from the body through shell command substitution, which collapses the
  title/body separator. We have hit this bug; do not repeat it.
- **ALWAYS** verify after each destructive-ish operation: `git status`, `git log -1`,
  `git show --no-patch --pretty=format:'%B' HEAD | head -5`, `gh release view`.

---

## Phase 1 — inspect state & detect new ccVersion

```bash
git status --short
git diff --stat HEAD
git log --oneline -5
git remote -v
```

The expected shape of `git status --short`:
- `M ` lines: upstream reworded something in an existing prompt.
- `D ` lines: upstream removed a prompt.
- `??` lines: upstream added a prompt.

If `git status --short` is empty, stop — preconditions failed (see above).

### Detect the new ccVersion

The ccVersion lives in each `.md` file's HTML-comment frontmatter as `ccVersion: X.Y.Z`.
After a bump, the modified files will carry the new version. Grab it from any `M` file:

```bash
grep -h '^ccVersion:' system-prompts/<one-of-the-M-files>.md
```

Or, to find the highest ccVersion in the whole tree (authoritative):

```bash
grep -h '^ccVersion:' system-prompts/*.md | sort -V | tail -1
```

Record the exact value — e.g. `2.1.117`. You'll reuse it in the commit title, tag name,
tag annotation, release title, release body, and zip filename. **One source of truth** —
derive every placeholder from this single string.

Also check what tags already exist for this version:

```bash
git tag --list "v<VERSION>*"
```

- No match → this is iteration 1 (tag will be `v<VERSION>-1`).
- `v<VERSION>-1` exists → iteration 2 (tag `v<VERSION>-2`). And so on.

---

## Phase 2 — re-apply un-nerfs

```bash
python scripts/apply-unnerfs.py
```

Read the full output. The `=== Summary ===` block at the end is the source of truth:

| Counter | Meaning | Action |
|---|---|---|
| `Rules applied` | Stock text matched, replaced with un-nerfed text. | Good. |
| `Rules skipped` | Un-nerfed text already present (idempotent re-run). | Good. |
| `Rules FAILED` | Neither stock nor un-nerfed text was found. | **Must resolve before continuing.** |
| `Missing files` | A rule targets a file that no longer exists. | Remove the rule (see below). |

**Common shape after a clean version bump**:
- Fresh stock just copied in → expect `Rules applied: N, Rules skipped: 0, FAILED: 0`.
- Script re-run after Phase 2 already succeeded → expect `Rules applied: 0, Rules skipped: N`.

### Resolving FAILs (iterate until zero)

For each `[FAIL]` entry in the report:

1. Read the named file in full. The report quotes the first ~200 chars of the expected
   stock — use that as a substring search to find where upstream moved the passage.
2. Open `scripts/apply-unnerfs.py`. Find the rule: `Rule(stock=..., unnerf=..., description=...)`
   under the filename key in the `RULES` dict.
3. Update the `stock` field to match the new upstream wording **byte-exactly**:
   - Preserve template literals (`${VAR}`), backticks, punctuation.
   - Preserve trailing whitespace (Python's multiline strings are picky here).
4. The `unnerf` field usually doesn't change — it's the target state. Only update it if
   upstream restructured the passage enough that the original replacement no longer fits.
5. Re-run `python scripts/apply-unnerfs.py`. Confirm the specific FAIL dropped off.
6. If another FAIL appears (possibly unmasked by the first fix), repeat. Loop until
   `Rules FAILED: 0`.

### Resolving Missing files

Caused by upstream deleting a file that a rule referenced.

1. Cross-check `git status --short` for a `D` entry matching the missing filename.
2. If confirmed deleted: open `apply-unnerfs.py`, remove the corresponding
   `"system-prompts/<filename>.md": [...]` entry from `RULES`. Note the removal in the
   sync commit body (the total rule count drops).
3. If NOT in `D` status: something else is wrong (stock copy-in incomplete?). Ask the user.

### Final gate for Phase 2

Re-run once more and confirm `Rules FAILED: 0` and `Missing files: 0`. Only then proceed.

---

## Phase 3 — bucket analysis on new and modified files

This is the **judgment step**. A careless sync skips it; a careful sync produces per-file
analysis that appears in both the commit body and the release body.

### The bucket taxonomy (restated — source of truth is the README)

| Bucket | Examples | Action |
|---|---|---|
| 1. Chat-brevity | "respond in 2-3 sentences", "terse one-liner is fine", length cap for classifier/structured output | **Keep stock.** |
| 2. Implementation-brevity | "simplest approach", "don't add abstractions", "don't gold-plate", "match the scope" | **Flip — add rule.** |
| 3. Process-brevity | "as quickly as possible", "don't explore more than necessary", "report back concisely" | **Flip — add rule.** |
| 4. Thoroughness | "think step by step", "consider edge cases", "check your work" | **Amplify** if weakened. |

### For each added (`??`) or modified (`M`) file

1. Read the whole file (the body, not just the diff — context matters).
2. List every directive-shaped sentence. Classify each into a bucket.
3. Decision:
   - Any bucket 2 / 3 directive → new rule warranted.
   - Bucket 4 directive that upstream weakened → new rule to restore strength.
   - Only bucket 1 or no directives → leave stock; document the "leave stock" decision.
4. Write a 2–4 sentence analysis summary for this file. **Don't silently skip**: the
   analysis ends up verbatim (or close to it) in the commit body and release body.

### Edge cases

- **Structured output / classifier-consumed prompts.** These often carry "one line" or
  similar length caps. If the cap is a tokenization/parsing aid (the downstream system
  can't handle variable-length output), it's bucket 1 — keep stock. If the cap is there
  purely to make the model respond tersely to humans, it's still bucket 1 but worth
  mentioning as a revisit candidate.
- **Autonomous-operation pragma.** Lines like "don't ask when a guess is cheaper than
  the round-trip" are about when to prompt the human, not about work quality — not
  bucket 3 process-brevity. Leave stock.
- **Prompts that are already thesis-aligned.** Some new prompts (narration-heavy,
  sanity-check-heavy) ship already pro-thoroughness. Document the alignment and move on.

### Adding a new rule

```python
# In scripts/apply-unnerfs.py, inside the RULES dict:
"system-prompts/<filename>.md": [
    Rule(
        stock="<exact text from new upstream, byte-exact>",
        unnerf="<thorough-over-brief replacement>",
        description="<short scannable label>",
    ),
    # ... more rules for the same file if needed
],
```

After editing:

```bash
python scripts/apply-unnerfs.py --dry-run --only <filename>.md   # preview
python scripts/apply-unnerfs.py --only <filename>.md             # actually apply
python scripts/apply-unnerfs.py --check                          # full tree must be idempotent
```

### Removing a rule

If upstream deleted a file the rule targeted, delete the whole entry from `RULES`.
Mention the rule-count change in the commit body.

---

## Phase 4 — compose and commit the sync

### Commit message template

```
sync prompts to Claude Code v<VERSION>

<One-paragraph summary: what upstream changed, net file count delta,
whether new un-nerf rules were added. Keep to 2-3 sentences.>

## Modified: <path to each M file>

<For each: what upstream changed. Whether any existing rule's stock text
drifted and was updated. Whether new un-nerfs were warranted (bucket
analysis, 2-3 sentences).>

## Deleted: <path to each D file>

<For each: confirmation that no rule in apply-unnerfs.py targeted it, OR
note that a rule was removed.>

## Added: <path to each new file>

<For each: one-paragraph description of what the prompt does. Then the
bucket analysis: which directives exist, which bucket each falls into,
and why the decision is to add a rule / leave stock.>

## Script state

`apply-unnerfs.py` run after the stock copy-in reports <N>/<N> rules
applied, <M> skipped, 0 failed, 0 missing — every pre-existing un-nerf
still matches the v<VERSION> stock text byte-exactly. <If rules changed:
"+K new rule(s) for <file>, -L removed rule(s) for <deleted file>.">
```

### Execute

```bash
# 1. Write message to temp file (NEVER inline via HEREDOC-in-$(...))
TMPMSG=$(mktemp --suffix=.txt)
```

Then use the `Write` tool to populate `$TMPMSG` with the full message — this is the
only way to guarantee blank-line preservation.

```bash
# 2. Stage files surgically
git add system-prompts/<file1>.md system-prompts/<file2>.md system-prompts/<file3>.md
# If rules changed:
git add scripts/apply-unnerfs.py

# 3. Verify staging
git status --short
# Should show exactly the files you intended. No surprises.

# 4. Commit
git commit -F "$TMPMSG"

# 5. Verify formatting — line 2 of the body should be BLANK
git show --no-patch --pretty=format:'%B' HEAD | cat -A | head -5
# Expected:
#   sync prompts to Claude Code v<VERSION>$
#   $                                          <-- this blank line is the critical check
#   <first paragraph>$
#   ...

# 6. Clean up message file
rm "$TMPMSG"
```

### If formatting is wrong (no blank line between title and body)

```bash
git reset --soft HEAD~1             # uncommit; keeps staging intact; ORIG_HEAD preserved
# Fix the temp file / rewrite $TMPMSG
git commit -F "$TMPMSG"              # re-commit
```

The old SHA remains recoverable via `git reflog` / `ORIG_HEAD` for ~90 days.

---

## Phase 5 — push the commit

If the user hasn't explicitly authorized pushes this session, confirm once here before
running. After confirmation:

```bash
git push origin master
```

Verify the push landed:

```bash
git fetch origin && git rev-list --left-right --count origin/master...HEAD
# Expect: 0<TAB>0
```

---

## Phase 6 — compute tag name and create annotated tag

Tag name: `v<VERSION>-<ITERATION>`, where `<ITERATION>` was determined in Phase 1.

### Tag annotation template

```
Un-nerfed system prompts for Claude Code v<VERSION> (iteration <N>)

Tagged snapshot aligned with Claude Code v<VERSION>, extracted via
tweakcc-fixed. Delta from v<PREV_TAG>: <1-2 sentences on file-count
delta and any notable upstream changes>. All un-nerfs remain idempotent
under scripts/apply-unnerfs.py (<N>/<N> rules applied against fresh
v<VERSION> stock, <M> skipped, 0 failed). <If rule changes: "Added K
new rule(s) for <file>." / "Removed L rule(s) for deleted <file>.">

The -<N> suffix is the iteration number against the same CC version;
future tweaks on top of v<VERSION> ship as v<VERSION>-<N+1>, etc.

Release asset system-prompts-v<VERSION>-<N>.zip contains the
<FILE_COUNT> .md files flat at the zip root, drop-in for
~/.tweakcc/system-prompts/.
```

### Execute

```bash
TMPTAG=$(mktemp --suffix=.txt)
# Use Write tool to populate $TMPTAG with the annotation

git tag -a "v<VERSION>-<N>" -F "$TMPTAG" HEAD

# Verify
git show --no-patch "v<VERSION>-<N>" | head -25

rm "$TMPTAG"
```

---

## Phase 7 — push the tag

```bash
git push origin "v<VERSION>-<N>"
```

---

## Phase 8 — build the zip asset

Always source the zip from the committed tree, not the working copy — this guarantees
the zip's contents match what the tag points at.

```bash
# Pick a stable temp path (NOT mktemp .zip — we need a specific filename for the asset)
ZIP_PATH="${TMPDIR:-/tmp}/system-prompts-v<VERSION>-<N>.zip"
rm -f "$ZIP_PATH"  # in case a stale one exists
git archive --format=zip -o "$ZIP_PATH" HEAD:system-prompts

# Verify: every entry is an .md file, flat at the root, count matches the repo
ls -la "$ZIP_PATH"
python - <<'PY'
import zipfile, os
path = os.environ.get("ZIP_PATH") or "<paste $ZIP_PATH>"
z = zipfile.ZipFile(path)
names = z.namelist()
md = [n for n in names if n.endswith(".md")]
non_md = [n for n in names if not n.endswith(".md")]
subdirs = [n for n in names if "/" in n]
print(f"total entries: {len(names)}")
print(f".md files   : {len(md)}")
print(f"non-md       : {non_md}")
print(f"with subdirs : {len(subdirs)}")
PY
```

Expected: `total == md count == <repo file count>`, `non-md` empty, `subdirs == 0`.
If subdirs > 0, the `HEAD:system-prompts` tree-reference went wrong — fix before
continuing.

Record two numbers for the release body:
- `<FILE_COUNT>` — number of `.md` files
- `<SIZE_KB>` — zip size in KB, rounded to nearest 10 (e.g. "~400 KB")

---

## Phase 9 — compose the release body

### Previous-tag lookup (for the "Changes since" link)

Before creating the new release:

```bash
gh release list --limit 1 --json tagName --jq '.[0].tagName'
# Returns the tag of the current latest release — use this as <PREV_TAG>.
```

### Release body template

```markdown
Un-nerfed Claude Code **v<VERSION>** system prompts — current latest.

- **Zip contents:** <FILE_COUNT> `.md` files at the zip root, ~<SIZE_KB> KB compressed.
- **Drop-in for:** `~/.tweakcc/system-prompts/`.
- **Full context:** [README at this tag](https://github.com/BenIsLegit/tweakcc-system-prompts-unnerfed/blob/v<VERSION>-<N>/README.md) — un-nerf thesis, before/after examples, install steps, post-bump re-apply workflow.
- **Install note:** requires [`tweakcc-fixed`](https://github.com/BenIsLegit/tweakcc-fixed) until upstream tweakcc catches up to Claude Code builds newer than v2.1.113.

## Changes since [`<PREV_TAG>`](https://github.com/BenIsLegit/tweakcc-system-prompts-unnerfed/releases/tag/<PREV_TAG>)

### Upstream (from Anthropic — Claude Code v<PREV_VERSION> → v<VERSION>)

These changes came from Anthropic's v<VERSION> build itself. This repo did **not** author any of them — they flow through verbatim from the stock `tweakcc` extraction, before any un-nerfing is applied.

- **+N file(s):** <new filename> — <one-sentence description of what the prompt does>
- **−N file(s):** <deleted filename> — removed entirely.
- **<change type> in `<filename>`:** <summary of the user-visible change, 1-2 sentences>

Net file count: <TOTAL>, <unchanged from <PREV_TAG> | +K from <PREV_TAG>>.

### This repo (un-nerf re-application only)

These are the only things this repo did on top of the upstream extraction. **No new prompt wording was authored here.** The un-nerfs are the same rules shipped in prior releases, replayed against the fresh v<VERSION> stock.

- **Un-nerfs re-applied cleanly:** `scripts/apply-unnerfs.py` reports **<N>/<N> rules applied, <M> skipped, 0 failed, 0 missing** against the fresh v<VERSION> stock. Every pre-existing un-nerf still matched upstream wording byte-exactly — no rule drift to chase, no rule body edits required.
- <EITHER:> **No new un-nerf rules added.** <Short bucket-analysis summary for new/modified files, 2-3 sentences, explaining why the decision was "leave stock". Link to the bucket taxonomy in the README.>
- <OR:> **Added <K> new un-nerf rule(s):** <bullet each new rule with filename + 1-sentence rationale>.
- <IF a rule was removed because a file was deleted:> **Removed <L> un-nerf rule(s)** because upstream deleted the targeted file(s): <list>.

---

**Version scheme.** `-<N>` is an iteration counter against the same Claude Code version.
```

### Filling the template

Every placeholder derives from data already gathered:
- `<VERSION>`, `<N>` — from Phase 1.
- `<PREV_TAG>`, `<PREV_VERSION>` — from `gh release list --limit 1`. `<PREV_VERSION>` is `<PREV_TAG>` minus the `-N` suffix.
- `<FILE_COUNT>`, `<SIZE_KB>` — from Phase 8.
- `<TOTAL>` — the same as `<FILE_COUNT>`.
- The per-file bullets in the Upstream section — from the Phase 3 analysis.
- The un-nerf summary in the This-repo section — from the Phase 3 analysis + Phase 2 script output.

---

## Phase 10 — publish the GitHub release

```bash
TMPBODY=$(mktemp --suffix=.md)
# Use Write tool to populate $TMPBODY with the composed release body

gh release create "v<VERSION>-<N>" "$ZIP_PATH" \
    --title "v<VERSION>-<N>" \
    --notes-file "$TMPBODY"

# The command outputs the release URL on success.
```

---

## Phase 11 — verify

```bash
gh release view "v<VERSION>-<N>" --json name,tagName,isDraft,isPrerelease,assets,body \
    --jq '{name, tagName, isDraft, isPrerelease, asset_count: (.assets | length), asset: .assets[0].name, body_preview: .body[0:500]}'

gh release list --limit 3
```

Expected:
- `name == tagName == "v<VERSION>-<N>"`.
- `isDraft: false`, `isPrerelease: false`.
- `asset_count: 1`, `asset: "system-prompts-v<VERSION>-<N>.zip"`.
- `body_preview` starts with `Un-nerfed Claude Code **v<VERSION>**`.
- In `gh release list`, the new release is line 1 with the **Latest** marker; the
  previous release has been demoted (no Latest marker).

If anything's off:
- Body wrong → `gh release edit v<VERSION>-<N> --notes-file <new-body-file>` (safe — doesn't touch tag/asset/flags).
- Asset wrong → `gh release upload v<VERSION>-<N> <new-zip> --clobber`.
- Wrong tag commit (very rare) → stop and escalate to the user; fixing this involves deleting the tag both locally and on origin, which is destructive.

---

## Phase 12 — clean up

```bash
rm -f "$ZIP_PATH" "$TMPBODY"
# $TMPMSG and $TMPTAG should already be cleaned up from their respective phases.
```

Verify the working tree is clean:

```bash
git status --short
# Should be empty (assuming no unrelated local changes).
```

---

## End-of-turn summary template

When all phases are complete, give the user a summary covering:

1. **Commit**: SHA, title, push status.
2. **Tag**: name, annotation preview, push status.
3. **Release**: URL, Latest flag confirmation, asset filename and size, body's two-subsection
   structure confirmed.
4. **apply-unnerfs.py state**: rule counts (applied / skipped / failed / missing), plus any
   rules added or removed in this sync.
5. **Per-file bucket analysis outcomes**: one line per new/modified file so the user can
   audit the judgment calls without re-reading the commit body.
6. **Anything unusual** — FAILs that required rule updates, edge cases in bucket analysis,
   new un-nerf rules that would benefit from a dedicated README example, etc.

Aim for a summary that a reader who didn't watch the session can read cold and know exactly
what shipped.
