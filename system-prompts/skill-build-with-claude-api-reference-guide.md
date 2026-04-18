<!--
name: 'Skill: Build with Claude API (reference guide)'
description: >-
  Template for presenting language-specific reference documentation with quick
  task navigation
ccVersion: 2.1.108
-->
## Reference Documentation

The relevant documentation for your detected language is included below in `<doc>` tags. Each tag has a `path` attribute showing its original file path. Use this to find the right section:

### Quick Task Reference

**Single text classification/summarization/extraction/Q&A:**
→ Refer to `{lang}/claude-api/README.md`

**Chat UI or real-time response display:**
→ Refer to `{lang}/claude-api/README.md` + `{lang}/claude-api/streaming.md`

**Long-running conversations (may exceed context window):**
→ Refer to `{lang}/claude-api/README.md` — see Compaction section
<!-- __G2__ -->
**Prompt caching / optimize caching / "why is my cache hit rate low":**
→ Refer to `shared/prompt-caching.md` + `{lang}/claude-api/README.md` (Prompt Caching section)

**Function calling / tool use / agents:**
→ Refer to `{lang}/claude-api/README.md` + `shared/tool-use-concepts.md` + `{lang}/claude-api/tool-use.md`

**Batch processing (non-latency-sensitive):**
→ Refer to `{lang}/claude-api/README.md` + `{lang}/claude-api/batches.md`

**File uploads across multiple requests:**
→ Refer to `{lang}/claude-api/README.md` + `{lang}/claude-api/files-api.md`

**Agent design (tool surface, context management, caching strategy):**
→ Refer to `shared/agent-design.md`

**Managed Agents (server-managed stateful agents):**
→ Refer to `shared/managed-agents-overview.md` and the rest of the `shared/managed-agents-*.md` files. For Python, TypeScript, and cURL, language-specific code examples live in `{lang}/managed-agents/README.md`. Java, Go, Ruby, and PHP also support the API — translate the calls using your SDK\'s patterns from `{lang}/claude-api.md`. C# does not currently have Managed Agents support; use raw HTTP from `curl/managed-agents.md` as a reference.

**Error handling:**
→ Refer to `shared/error-codes.md`

**Latest docs via WebFetch:**
→ Refer to `shared/live-sources.md` for URLs
