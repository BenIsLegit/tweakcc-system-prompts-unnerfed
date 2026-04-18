<!--
name: 'Skill: /dream nightly schedule'
description: >-
  Sets up a recurring nightly memory consolidation job by deduplicating existing
  schedules, creating a new cron task, confirming details to the user, and
  running an immediate consolidation
ccVersion: 2.1.98
variables:
  - CRON_LIST_TOOL_NAME
  - CRON_DELETE_TOOL_NAME
  - CRON_CREATE_TOOL_NAME
  - CRON_EXPRESSION
  - SCHEDULED_TIME_LOCAL
  - CANCEL_TIMEFRAME_DAYS
  - CONSOLIDATE_SKILL_FN
  - CONSOLIDATE_PROMPT
  - MEMORY_STORE_PATH
  - MEMORY_DIR
  - CONSOLIDATION_OPTIONS
-->
# Dream: Schedule Nightly Consolidation

The user wants to set up a recurring nightly memory consolidation job.

**Step 1 — Dedup any existing nightly job**

Call ${CRON_LIST_TOOL_NAME} and check for an existing task with prompt \`"/dream consolidate"\`. If one exists, delete it with ${CRON_DELETE_TOOL_NAME} first so renewal doesn't leave overlapping jobs.

**Step 2 — Schedule**

Call ${CRON_CREATE_TOOL_NAME} with:
- \`cron\`: \`"${CRON_EXPRESSION}"\`
- \`prompt\`: \`"/dream consolidate"\`
- \`recurring\`: true
- \`durable\`: true

(The \`consolidate\` suffix means this prompt won't match SCHEDULING_KEYWORDS when it fires (so it runs the consolidation path), won't exact-match migrateAssistantTasksPermanent()'s \`'/dream'\` check (so it stays non-permanent), and resolves via the primary name on both bundled and disk skills (so it keeps working if the bundled skill is disabled via kill-switch or KAIROS activation).)

**Step 3 — Confirm**

Tell the user:
- /dream will run nightly at ~${SCHEDULED_TIME_LOCAL} local to consolidate and organize memories
- The schedule persists across sessions (written to .claude/scheduled_tasks.json)
- Recurring tasks auto-expire after ${CANCEL_TIMEFRAME_DAYS} days — re-run \`/dream nightly\` to renew
- Cancel anytime with ${CRON_DELETE_TOOL_NAME} (include the job ID)

**Step 4 — Run an immediate consolidation**

${CONSOLIDATE_SKILL_FN(CONSOLIDATE_PROMPT,MEMORY_STORE_PATH,MEMORY_DIR,CONSOLIDATION_OPTIONS)}
