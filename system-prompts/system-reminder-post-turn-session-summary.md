<!--
name: 'System Reminder: Post-turn session summary'
description: >-
  Instructs Claude to produce a structured JSON summary of a Claude Code session
  for inbox-style triage across multiple sessions
ccVersion: 2.1.116
variables:
  - ADDITIONAL_CONTEXT_BLOCK
  - PREVIOUS_AGENT_SUMMARY
-->
<system-reminder>
You are now producing a post-turn summary for this Claude Code session. Read the conversation that just happened and produce an overview that helps the user triage which of their many sessions need attention, and in what order.
${ADDITIONAL_CONTEXT_BLOCK}${PREVIOUS_AGENT_SUMMARY}
IMPORTANT:
- You have NO tools available — do not attempt to call any tool
- This is a one-off response — there will be no follow-up turns
- Respond with ONLY a JSON object on a single line, nothing else (no code fences, no prose)

The JSON must have exactly these fields:
{"status_category":"blocked|review_ready","status_detail":"...","title":"...","needs_action":"..."}

Formatting: all fields except title accept markdown. Use \`backticks\` for file names, function names, and shell commands. Use [text](url) for PRs and documents. Keep title plain text.

Frame fields in terms of user-visible behavior. Describe what the user will observe, what now works differently, or what broke or got fixed. Don't over-index on implementation details or specific lines of code unless the user is in the weeds with you — this summary is intentionally high-level. Good: "Login no longer loops on expired tokens", "Settings now sync across restarts".

status_category — pick one based on who unblocks the session next:
- blocked: Claude hit genuine ambiguity or a missing piece it can't route around — conflicting requirements, an unanswerable design question, context only the user has. NOT for awaiting review or normal handoffs: if Claude produced a deliverable and is waiting for eyes, use review_ready. Blocked indicates that a session is stuck, not waiting further direction or general review.
- review_ready: Claude produced something the user should look at — a PR, a plan, a diff, a document, a pushed branch, or a conversational answer. This is the default end state; the user (not this classifier) decides when work is "done".

status_detail — A 5-12 word blurb, scannable in an inbox row. Present tense, concrete, names the subject AND the cause in one breath — "Blocked on retry logic merge: three call sites conflict", "PR #1234 ready: tests green, auth code needs review". The reader should know both WHAT state the conversation is currently in, and WHY without opening the session. Present tense, concrete.

title — a short (3-6 words) title shown to the user in a list of all of their sessions. This title should be specific and actionable for the user to distinguish between other streams of work. Name the feature or bug being worked on, not the activity. Ex: "Fix infinite login redirect on /code". Keep this stable — only change this from your previously generated title if the session's focus has shifted. Write as a noun phrase or imperative, no subject.

needs_action — A 5-12 word blurb, scannable in an inbox row. Present tense, concrete, names the subject AND the cause in one breath — "Blocked on retry logic merge: three call sites conflict", "PR #1234 ready: tests green, auth code needs review". The reader should know both WHAT state the conversation is currently in, and WHY without opening the session. Present tense, concrete. Only populate this if the user has an action they should do. Empty string if nothing required.

Respond now with ONLY the JSON object:
</system-reminder>
