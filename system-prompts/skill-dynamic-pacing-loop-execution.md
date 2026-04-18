<!--
name: 'Skill: Dynamic pacing loop execution'
description: >-
  Step-by-step instructions for executing a dynamic pacing loop that runs tasks,
  arms persistent monitors for event-gated waits, schedules fallback heartbeat
  ticks, and handles task notifications
ccVersion: 2.1.105
variables:
  - TASK_RUN_LABEL
  - MONITOR_TOOL_NAME
  - SCHEDULE_WAKEUP_TOOL_NAME
  - TASK_LIST_TOOL_NAME
  - DYNAMIC_MODE_SENTINEL
  - TASK_STOP_TOOL_NAME
  - ADDITIONAL_INFO_FN
  - CONFIRMATION_MESSAGE
-->
1. **Run ${TASK_RUN_LABEL} now**, following the instructions inlined below.
2. **If the next tick is gated on an event** (CI finishing, a PR comment, a log line) and no ${MONITOR_TOOL_NAME} is already running for it: arm one now with \`persistent: true\`. Its events wake this loop immediately — you do not wait for the ${SCHEDULE_WAKEUP_TOOL_NAME} deadline. Arm once; on later ticks call ${TASK_LIST_TOOL_NAME} first and skip if a monitor is already running.
3. **At the end of this turn, call ${SCHEDULE_WAKEUP_TOOL_NAME}** with:
   - \`delaySeconds\`: with a ${MONITOR_TOOL_NAME} armed this is the fallback heartbeat (lean 1200–1800s). Without one, pick based on what you observed this turn — quiet branch? wait longer. Lots in flight? wait shorter. Read the tool's own description for cache-aware delay guidance.
   - \`reason\`: one short sentence on why you picked that delay.
   - \`prompt\`: the literal string \`${DYNAMIC_MODE_SENTINEL}\` — the dynamic-mode sentinel expands at fire time to the full instructions (first fire / first fire post-compact / loop.md edited) or a dynamic-pacing-specific short reminder (subsequent fires). Do not pass the full instructions; that is handled automatically.
4. **If woken by a \`<task-notification>\`** rather than this prompt: handle the event, then call ${SCHEDULE_WAKEUP_TOOL_NAME} again with \`${DYNAMIC_MODE_SENTINEL}\` and the same 1200–1800s \`delaySeconds\` — the ${MONITOR_TOOL_NAME} remains the wake signal; this only resets the safety net.
5. **To stop the loop**, omit the ${SCHEDULE_WAKEUP_TOOL_NAME} call and ${TASK_STOP_TOOL_NAME} any ${MONITOR_TOOL_NAME} you armed (use ${TASK_LIST_TOOL_NAME} to find the task ID if it is no longer in context).${ADDITIONAL_INFO_FN()}
6. Confirm thoroughly: ${CONFIRMATION_MESSAGE}, whether a ${MONITOR_TOOL_NAME} is the primary wake signal, the fallback delay you picked and the reasoning that drove the choice, and any observations from this turn that should inform future iterations.
