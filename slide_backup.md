# Slide → Code Reference Map

Backing references for `build_deck_v6.js`, mapped against
`cc-code/restored-src/src` (the restored Claude Code source tree).

Slide numbers follow the comment markers in the deck generator
(`// 1 · TITLE` … `// 22 · CLOSE`), not the on-slide section badges.

Link paths are repo-root-relative, matching the convention in `cc_detail.md`
and `cc_code.md`.

---

# Slides with direct code backing

## Slide 3 · Vocabulary — the table is a map of the tree

| Row | Code |
|---|---|
| Model | [claude.ts:752](cc-code/restored-src/src/services/api/claude.ts#L752) `queryModelWithStreaming` |
| Harness | the whole `src/` tree |
| Agent | [query.ts:241](cc-code/restored-src/src/query.ts#L241) `queryLoop` |
| Context — "rebuilt every turn" | [query.ts:365-467](cc-code/restored-src/src/query.ts#L365-L467) — `messagesForQuery` is reassembled at the top of *every* iteration |
| Tool | [Tool.ts:362](cc-code/restored-src/src/Tool.ts#L362) interface, [`buildTool`:783](cc-code/restored-src/src/Tool.ts#L783) |
| MCP server | [services/mcp/](cc-code/restored-src/src/services/mcp/), `normalization.ts` (`mcp__server__tool` prefixing) |
| Skill | [skills/loadSkillsDir.ts](cc-code/restored-src/src/skills/loadSkillsDir.ts), [tools/SkillTool/](cc-code/restored-src/src/tools/SkillTool/) |
| Subagent | [runAgent.ts:248](cc-code/restored-src/src/tools/AgentTool/runAgent.ts#L248) |
| Task | [tasks/types.ts](cc-code/restored-src/src/tasks/types.ts), [utils/task/framework.ts](cc-code/restored-src/src/utils/task/framework.ts) |
| Hook | [`HOOK_EVENTS` coreTypes.ts:25](cc-code/restored-src/src/entrypoints/sdk/coreTypes.ts#L25) |

## Slide 6 · Prompt cache — the strongest theory→code link in the deck

- "Exact prefix match, tools → system → messages" ←
  [`assembleToolPool` tools.ts:345-357](cc-code/restored-src/src/tools.ts#L345-L357).
  **This comment is the slide, verbatim from the source:** *"Sort each partition for
  prompt-cache stability, keeping built-ins as a contiguous prefix. The server's cache
  policy places a global cache breakpoint after the last prefix-matched built-in tool; a
  flat sort would interleave MCP tools into built-ins and invalidate all downstream…"*
- "4 breakpoints" ← [`addCacheBreakpoints` claude.ts:3063](cc-code/restored-src/src/services/api/claude.ts#L3063),
  and [:3078](cc-code/restored-src/src/services/api/claude.ts#L3078) *"Exactly one
  message-level cache_control marker per request."*
- TTL / 1-hour vs 5-minute ← [`getCacheControl` claude.ts:358](cc-code/restored-src/src/services/api/claude.ts#L358),
  with [:404](cc-code/restored-src/src/services/api/claude.ts#L404) on mid-session overage
  flipping the TTL
- "Tool definitions are deploy artefacts" ← same tools.ts:354-357 comment
- "Volatile content goes last" ← [context.ts:36/116/155](cc-code/restored-src/src/context.ts#L116)
  — `getGitStatus`, `getSystemContext`, `getUserContext` are all `memoize`d *specifically*
  so they don't churn the prefix
- Observability ← [promptCacheBreakDetection.ts](cc-code/restored-src/src/services/api/promptCacheBreakDetection.ts)
  `notifyCompaction`; usage fields at [query.ts:489-491](cc-code/restored-src/src/query.ts#L489-L491)
- Bonus: `skipCacheWrite` is a real per-query param — [query.ts:696](cc-code/restored-src/src/query.ts#L696)

## Slide 7 · The loop

- `while True:` ← [query.ts:307](cc-code/restored-src/src/query.ts#L307)
- Step 1 Shape context ← [query.ts:365-467](cc-code/restored-src/src/query.ts#L365-L467)
- Step 2 Call model ← [query.ts:659](cc-code/restored-src/src/query.ts#L659) `deps.callModel`
- Step 3 Stream & collect ← [query.ts:708-863](cc-code/restored-src/src/query.ts#L708-L863)
- Step 4 Tool calls ← [query.ts:1366-1408](cc-code/restored-src/src/query.ts#L1366-L1408)
- **"Exit is typed, not thrown: ten terminal reasons"** ←
  [query/transitions.ts](cc-code/restored-src/src/query/transitions.ts) `Terminal`.
  The `return` sites: `blocking_limit`(646), `image_error`(977), `model_error`(996),
  `aborted_streaming`(1051), `prompt_too_long`(1175/1182), `completed`(1264/1357),
  `stop_hook_prevented`(1279), `aborted_tools`(1515), `hook_stopped`(1520),
  `max_turns`(1711). **Ten. The number is exact.**
- "UI-agnostic, I/O injected as callbacks" ← [`ToolUseContext` Tool.ts:158](cc-code/restored-src/src/Tool.ts#L158)
  + [query/deps.ts](cc-code/restored-src/src/query/deps.ts) `productionDeps`
  + [query/config.ts](cc-code/restored-src/src/query/config.ts)
- "One loop serves CLI, headless and SDK" ← [REPL.tsx:2793](cc-code/restored-src/src/screens/REPL.tsx#L2793)
  vs [QueryEngine.ts:209](cc-code/restored-src/src/QueryEngine.ts#L209)

## Slide 9 · Context

- Four sources ← [`fetchSystemPromptParts` queryContext.ts:44](cc-code/restored-src/src/utils/queryContext.ts#L44)
  · [`getUserContext` context.ts:155](cc-code/restored-src/src/context.ts#L155)
  · [`getSystemContext` context.ts:116](cc-code/restored-src/src/context.ts#L116)
  · [`getAttachmentMessages` attachments.ts](cc-code/restored-src/src/utils/attachments.ts)
- "Memoised per conversation so it doesn't churn the cache prefix" ← the `memoize(`
  wrappers at context.ts:36/116/155
- Assembly point ← [query.ts:449-451](cc-code/restored-src/src/query.ts#L449-L451)
  `appendSystemContext`, [query.ts:660](cc-code/restored-src/src/query.ts#L660) `prependUserContext`
- Tier table ←
  - Turn-local = [query.ts:365](cc-code/restored-src/src/query.ts#L365) the messages array
  - Session = [sessionStorage.ts:202](cc-code/restored-src/src/utils/sessionStorage.ts#L202)
    `getTranscriptPath` JSONL
  - Project = [memdir/](cc-code/restored-src/src/memdir/) + CLAUDE.md via `getUserContext`
  - Organisational = [skills/loadSkillsDir.ts](cc-code/restored-src/src/skills/loadSkillsDir.ts)
    + [tools.ts](cc-code/restored-src/src/tools.ts)
- "No separate personality config" ← [utils/systemPromptType.ts](cc-code/restored-src/src/utils/systemPromptType.ts)
  `asSystemPrompt` — sections concatenated at runtime

## Slide 11 · Compaction — near-perfect 1:1

| Slide stage | Code |
|---|---|
| 1 Tool-result budget | [query.ts:379-394](cc-code/restored-src/src/query.ts#L379-L394) `applyToolResultBudget` |
| 2 Snip | [query.ts:400-410](cc-code/restored-src/src/query.ts#L400-L410), `snipCompact.ts` |
| 3 Microcompact | [query.ts:413-426](cc-code/restored-src/src/query.ts#L413-L426), [`COMPACTABLE_TOOLS` microCompact.ts:41-50](cc-code/restored-src/src/services/compact/microCompact.ts#L41-L50) |
| 4 Context collapse | [query.ts:440-447](cc-code/restored-src/src/query.ts#L440-L447) |
| 5 Autocompact **(model call)** | [query.ts:453-467](cc-code/restored-src/src/query.ts#L453-L467) → [`autoCompactIfNeeded` autoCompact.ts:241](cc-code/restored-src/src/services/compact/autoCompact.ts#L241) |

- "Cheapest first, heavier becomes a no-op" ← [query.ts:428-439](cc-code/restored-src/src/query.ts#L428-L439)
  — the collapse-before-autocompact comment says exactly this
- "Microcompact blanks CONTENT but keeps structure" ←
  [`TIME_BASED_MC_CLEARED_MESSAGE` microCompact.ts:36](cc-code/restored-src/src/services/compact/microCompact.ts#L36)
  = `'[Old tool result content cleared]'`
- "Collapse is a read-time projection with summaries in a side store" ←
  [query.ts:428-439](cc-code/restored-src/src/query.ts#L428-L439) verbatim
- **The 1,279 / 3,272 / 250K numbers** ← [autoCompact.ts:67-70](cc-code/restored-src/src/services/compact/autoCompact.ts#L67-L70),
  cap at [:70](cc-code/restored-src/src/services/compact/autoCompact.ts#L70)
  `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`, enforcement
  [:257-265](cc-code/restored-src/src/services/compact/autoCompact.ts#L257-L265)
- "It does not back off, it stops" ← [autoCompact.ts:264](cc-code/restored-src/src/services/compact/autoCompact.ts#L264)
  — plain `return { wasCompacted: false }`
- Proactive ← query.ts:453 · Reactive ← [query.ts:1119-1166](cc-code/restored-src/src/query.ts#L1119-L1166)
- Thresholds if anyone asks: [`AUTOCOMPACT_BUFFER_TOKENS = 13_000`](cc-code/restored-src/src/services/compact/autoCompact.ts#L62),
  [`MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000`](cc-code/restored-src/src/services/compact/autoCompact.ts#L30)

## Slide 12 · Exception handling

- "Errors become values" ← [toolExecution.ts:469-486](cc-code/restored-src/src/services/tools/toolExecution.ts#L469-L486)
- "Orphan tool_use makes the NEXT request fail" ←
  [`yieldMissingToolResultBlocks` query.ts:123-149](cc-code/restored-src/src/query.ts#L123-L149)
- **The four defence sites** ← query.ts:[900](cc-code/restored-src/src/query.ts#L900) (fallback)
  · [984](cc-code/restored-src/src/query.ts#L984) (error)
  · [1025](cc-code/restored-src/src/query.ts#L1025) (abort)
  · [StreamingToolExecutor.ts:153-205](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L153-L205) (siblings)
  · [query.ts:1015-1023](cc-code/restored-src/src/query.ts#L1015-L1023) (drain-on-abort)
- Withhold ladder, 3 classes ← [query.ts:788-825](cc-code/restored-src/src/query.ts#L788-L825);
  the "SDK consumers kill the session" rationale is [query.ts:166-172](cc-code/restored-src/src/query.ts#L166-L172)
- Withhold → Recover → Escalate → Surface&stop ←
  [1089-1117](cc-code/restored-src/src/query.ts#L1089-L1117) →
  [1119-1166](cc-code/restored-src/src/query.ts#L1119-L1166) →
  [1199-1252](cc-code/restored-src/src/query.ts#L1199-L1252) →
  [1168-1183](cc-code/restored-src/src/query.ts#L1168-L1183)
- **The verbatim quote on the slide** ← [query.ts:1168-1172](cc-code/restored-src/src/query.ts#L1168-L1172).
  Accurate word for word.
- "Bounded at 3" ← [`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3` query.ts:164](cc-code/restored-src/src/query.ts#L164)
- Transient/semantic/bugs layering ← [withRetry.ts:170](cc-code/restored-src/src/services/api/withRetry.ts#L170)
  / `FallbackTriggeredError` at [query.ts:894](cc-code/restored-src/src/query.ts#L894) / everything else
- "3-10× gateway load" ← [withRetry.ts:58-61](cc-code/restored-src/src/services/api/withRetry.ts#L58-L61)
- Bounded shutdown ← [gracefulShutdown.ts:414-419](cc-code/restored-src/src/utils/gracefulShutdown.ts#L414-L419)

## Slide 13 · Dead loops — all six, all real

| Cause | Code |
|---|---|
| 1 Error→recovery→same error | [query.ts:1290-1297](cc-code/restored-src/src/query.ts#L1290-L1297) *(the "burning thousands of API calls" quote)*, [1168-1172](cc-code/restored-src/src/query.ts#L1168-L1172), [1258-1265](cc-code/restored-src/src/query.ts#L1258-L1265) |
| 2 Unbounded retry | [autoCompact.ts:67-70](cc-code/restored-src/src/services/compact/autoCompact.ts#L67-L70) |
| 3 Self-recursion | [autoCompact.ts:169-183](cc-code/restored-src/src/services/compact/autoCompact.ts#L169-L183), [query.ts:600-603](cc-code/restored-src/src/query.ts#L600-L603) |
| 4 Missing caps | [query.ts:164](cc-code/restored-src/src/query.ts#L164), [1705-1712](cc-code/restored-src/src/query.ts#L1705-L1712), [1506-1514](cc-code/restored-src/src/query.ts#L1506-L1514) (abort path!), `errorUtils.ts:51`, `sessionIngress.ts:308` |
| 5 Reactive feedback | [REPL.tsx:2631-2633](cc-code/restored-src/src/screens/REPL.tsx#L2631-L2633) (tick→error→tick), [skillChangeDetector.ts](cc-code/restored-src/src/utils/skills/skillChangeDetector.ts) `RELOAD_DEBOUNCE_MS=300`, [attachments.ts:1056](cc-code/restored-src/src/utils/attachments.ts#L1056) (0ms delay) |
| 6 Real deadlock | [remoteManagedSettings/index.ts:64](cc-code/restored-src/src/services/remoteManagedSettings/index.ts#L64), [policyLimits/index.ts:68](cc-code/restored-src/src/services/policyLimits/index.ts#L68), [skillChangeDetector.ts:52-60](cc-code/restored-src/src/utils/skills/skillChangeDetector.ts#L52-L60) (Bun `__ulock_wait2`) |

- **Latch reset policy** — the best line on the slide, and the code proves it:
  [`State` query.ts:204-217](cc-code/restored-src/src/query.ts#L204-L217);
  preserved at [1297](cc-code/restored-src/src/query.ts#L1297),
  reset at [1721](cc-code/restored-src/src/query.ts#L1721)
- Provenance ← [constants/querySource.ts](cc-code/restored-src/src/constants/querySource.ts)
  + autoCompact.ts:171
- Checklist item 1 "named transitions" ← [query/transitions.ts](cc-code/restored-src/src/query/transitions.ts)
  `Continue`; item 8 "typed exit" ← `Terminal`

## Slide 14 · Concurrency

- **"Zero worker_threads, zero new Worker()"** — grep run against the tree.
  **Confirmed, zero hits.** Safe to say on stage.
- Three layers ← async = query.ts generators · subprocess = BashTool / `ripgrep.ts` /
  `hooks.ts` / `LSPClient.ts` · process = [utils/swarm/backends/](cc-code/restored-src/src/utils/swarm/backends/)
- Cancellation tree ← [abortController.ts](cc-code/restored-src/src/utils/abortController.ts)
  `createChildAbortController`; sibling [StreamingToolExecutor.ts:45-48](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L45-L48);
  per-tool [:301-303](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L301-L303)
- "Aborting a child never touches the parent" ← abortController.ts docstring, verbatim
- The escape hatch ← [StreamingToolExecutor.ts:304-318](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L304-L318)
  (cites regression #21056)
- WeakRef leak ← abortController.ts `propagateAbort` / `removeAbortHandler`
- "Abort reason is a typed protocol" ← [`getAbortReason`:210-231](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L210-L231)
  + [`interruptBehavior` Tool.ts:407-414](cc-code/restored-src/src/Tool.ts#L407-L414)
- **"Async lock, never sync"** ← [teammateMailbox.ts:31-41](cc-code/restored-src/src/utils/teammateMailbox.ts#L31-L41).
  The comment literally says *"The sync lockSync API blocked the event loop."*

## Slide 15 · Multi-agent

- L1 ← [StreamingToolExecutor.ts:129-151](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L129-L151);
  fail-closed default [Tool.ts:750](cc-code/restored-src/src/Tool.ts#L750)
- "Evaluated per INPUT, not per tool type" ← [`isConcurrencySafe(input)` Tool.ts:402](cc-code/restored-src/src/Tool.ts#L402)
  + [StreamingToolExecutor.ts:104-113](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L104-L113)
- L2, "N subagents fan out in parallel" ← [AgentTool.tsx:1273](cc-code/restored-src/src/tools/AgentTool/AgentTool.tsx#L1273)
  returns `true`
- L3 ← [AgentTool.tsx:87](cc-code/restored-src/src/tools/AgentTool/AgentTool.tsx#L87);
  **severed abort tree** [:694](cc-code/restored-src/src/tools/AgentTool/AgentTool.tsx#L694);
  **promotion race** [:808-890](cc-code/restored-src/src/tools/AgentTool/AgentTool.tsx#L808-L890);
  `PROGRESS_THRESHOLD_MS = 2000` at [:63](cc-code/restored-src/src/tools/AgentTool/AgentTool.tsx#L63)
- L4 ← [utils/swarm/](cc-code/restored-src/src/utils/swarm/),
  [SendMessageTool.ts](cc-code/restored-src/src/tools/SendMessageTool/SendMessageTool.ts)
  (broadcast `@team` :260, request :114, shutdown :276),
  [teammateMailbox.ts](cc-code/restored-src/src/utils/teammateMailbox.ts)
- **"Chain id and depth on every event"** ← [query.ts:347-355](cc-code/restored-src/src/query.ts#L347-L355)
  — threaded into ~15 `logEvent` calls
- "Smaller tool pool = governance boundary" ← `runAgent` filtered tools
  + [`getTools` tools.ts](cc-code/restored-src/src/tools.ts)

## Slide 16 · Governance

- Gate ← [`hasPermissionsToUseTool` permissions.ts:473](cc-code/restored-src/src/utils/permissions/permissions.ts#L473),
  [hooks/useCanUseTool.tsx](cc-code/restored-src/src/hooks/useCanUseTool.tsx)
- **Deny > ask > allow** ← [`getDenyRules`:213](cc-code/restored-src/src/utils/permissions/permissions.ts#L213),
  [`getAskRules`:223](cc-code/restored-src/src/utils/permissions/permissions.ts#L223),
  [`getAllowRules`:122](cc-code/restored-src/src/utils/permissions/permissions.ts#L122)
- Modes ← [PermissionMode.ts:50-71](cc-code/restored-src/src/utils/permissions/PermissionMode.ts#L50-L71)
  — `default` / `plan` / `acceptEdits` / `bypassPermissions`
- Mode read inside the loop ← [query.ts:570-578](cc-code/restored-src/src/query.ts#L570-L578)
  (and `plan` mode even changes the model)
- "Batch needs a policy for ask" ← [yoloClassifier.ts:100](cc-code/restored-src/src/utils/permissions/yoloClassifier.ts#L100)
  `getDefaultExternalAutoModeRules`, [:125](cc-code/restored-src/src/utils/permissions/yoloClassifier.ts#L125);
  `isNonInteractiveSession` at [query.ts:675](cc-code/restored-src/src/query.ts#L675)
- Hooks ← [`HOOK_EVENTS` coreTypes.ts:25](cc-code/restored-src/src/entrypoints/sdk/coreTypes.ts#L25);
  wired at [query.ts:1001](cc-code/restored-src/src/query.ts#L1001),
  [:1174](cc-code/restored-src/src/query.ts#L1174), [:1267](cc-code/restored-src/src/query.ts#L1267)
- Read ≠ write ≠ send ← [`isReadOnly` / `isDestructive` Tool.ts:404-406](cc-code/restored-src/src/Tool.ts#L404-L406)

## Slide 17 · Skills explosion

- "Description text is rent" ← [SkillTool/prompt.ts:22-30](cc-code/restored-src/src/tools/SkillTool/prompt.ts#L22-L30)
  — *"verbose whenToUse strings waste turn-1 cache_creation tokens without improving match rate"*
- **1% and 250 chars** ← `SKILL_BUDGET_CONTEXT_PERCENT = 0.01`,
  `MAX_LISTING_DESC_CHARS = 250`, same lines
- Defer loading ← [utils/toolSearch.ts](cc-code/restored-src/src/utils/toolSearch.ts) whole file
- **"~10% of the window, measured cost not tool count"** ←
  [`DEFAULT_AUTO_TOOL_SEARCH_PERCENTAGE = 10` toolSearch.ts:46](cc-code/restored-src/src/utils/toolSearch.ts#L46)
  + `checkAutoThreshold` (token API, char fallback)
- "Name and one line" ← [Tool.ts:373](cc-code/restored-src/src/Tool.ts#L373)
  *"One-line capability phrase used by ToolSearch for keyword matching"*
- "Discovery must survive compaction" ← [`extractDiscoveredToolNames` toolSearch.ts:495-556](cc-code/restored-src/src/utils/toolSearch.ts#L495-L556)
  + `preCompactDiscoveredTools`
- "Announce as a delta" ← [`getDeferredToolsDelta` toolSearch.ts:600-680](cc-code/restored-src/src/utils/toolSearch.ts#L600-L680)
- **The `paths:` YAML block on the slide is real frontmatter** ← parsed at
  [loadSkillsDir.ts:159-178](cc-code/restored-src/src/skills/loadSkillsDir.ts#L159-L178),
  pool split at [:771-796](cc-code/restored-src/src/skills/loadSkillsDir.ts#L771-L796),
  activation at [`activateConditionalSkillsForPaths`:997-1056](cc-code/restored-src/src/skills/loadSkillsDir.ts#L997-L1056)
- **The 97% number** ← [query.ts:323-335](cc-code/restored-src/src/query.ts#L323-L335)
- "250-570ms hidden in a 2-30s turn" ← [query.ts:331-335](cc-code/restored-src/src/query.ts#L331-L335)
  + [:1620-1628](cc-code/restored-src/src/query.ts#L1620-L1628)

## Slide 18 · Long-running tasks

- Disk from byte zero ← [`DiskTaskOutput` diskOutput.ts:97](cc-code/restored-src/src/utils/task/diskOutput.ts#L97);
  `MAX_TASK_OUTPUT_BYTES = 5GB` [:30](cc-code/restored-src/src/utils/task/diskOutput.ts#L30);
  truncation marker [:117-120](cc-code/restored-src/src/utils/task/diskOutput.ts#L117-L120)
- Read by offset ← [`getTaskOutputDelta`:302-334](cc-code/restored-src/src/utils/task/diskOutput.ts#L302-L334)
- **"Exactly one owner per notification"** ← [framework.ts:199-203](cc-code/restored-src/src/utils/task/framework.ts#L199-L203)
  — the dual-delivery comment
- Poll backstop ← [`POLL_INTERVAL_MS = 1000` framework.ts:22](cc-code/restored-src/src/utils/task/framework.ts#L22)
- Priority queue ← [messageQueueManager.ts:126-143](cc-code/restored-src/src/utils/messageQueueManager.ts#L126-L143);
  drained mid-turn [query.ts:1566-1590](cc-code/restored-src/src/query.ts#L1566-L1590);
  agent-scoped [:1560-1578](cc-code/restored-src/src/query.ts#L1560-L1578)
- **"Absence of a status field is load-bearing"** ← [LocalShellTask.tsx:79-89](cc-code/restored-src/src/tasks/LocalShellTask/LocalShellTask.tsx#L79-L89)
  — the comment says exactly that
- Idempotent atomic completion ← [LocalShellTask.tsx:105-120](cc-code/restored-src/src/tasks/LocalShellTask/LocalShellTask.tsx#L105-L120)
- **The watchdog, all four steps** ← [`startStallWatchdog` LocalShellTask.tsx:46-104](cc-code/restored-src/src/tasks/LocalShellTask/LocalShellTask.tsx#L46-L104);
  `looksLikePrompt` at [:36-42](cc-code/restored-src/src/tasks/LocalShellTask/LocalShellTask.tsx#L36-L42);
  the actionable `echo y | command` remediation at [:88](cc-code/restored-src/src/tasks/LocalShellTask/LocalShellTask.tsx#L88);
  `timer.unref()` at [:99](cc-code/restored-src/src/tasks/LocalShellTask/LocalShellTask.tsx#L99)
- Task tools ← [tools/TaskCreateTool/](cc-code/restored-src/src/tools/TaskCreateTool/) … ;
  [`TaskOutputTool.isConcurrencySafe → true`:160](cc-code/restored-src/src/tools/TaskOutputTool/TaskOutputTool.tsx#L160)

## Slide 22 · Close — each clause has a file

| "…becomes…" | Code |
|---|---|
| Errors → typed messages | [toolExecution.ts:475](cc-code/restored-src/src/services/tools/toolExecution.ts#L475) |
| Continuation → named transition | [query/transitions.ts](cc-code/restored-src/src/query/transitions.ts) |
| Tool safety → declared predicate | [Tool.ts:402](cc-code/restored-src/src/Tool.ts#L402) |
| Skill relevance → a path | [loadSkillsDir.ts:159](cc-code/restored-src/src/skills/loadSkillsDir.ts#L159) |
| Task completion → protocol field | [LocalShellTask.tsx:79-89](cc-code/restored-src/src/tasks/LocalShellTask/LocalShellTask.tsx#L79-L89) |

Bounds in the speaker note: 3 ← [query.ts:164](cc-code/restored-src/src/query.ts#L164)
· 3 ← [autoCompact.ts:70](cc-code/restored-src/src/services/compact/autoCompact.ts#L70)
· 1% ← [SkillTool/prompt.ts:22](cc-code/restored-src/src/tools/SkillTool/prompt.ts#L22)
· 10% ← [toolSearch.ts:46](cc-code/restored-src/src/utils/toolSearch.ts#L46)
· 45s ← `STALL_THRESHOLD_MS`
· 5s ← [gracefulShutdown.ts:414](cc-code/restored-src/src/utils/gracefulShutdown.ts#L414)

---

# Slides with partial or no in-repo backing

- **Slide 10 · Context management** — the four platform primitives are *server-side API
  features*, so there's no code for them here. But three have client-side analogues worth
  pointing at: context editing ≈ [microCompact.ts:36-50](cc-code/restored-src/src/services/compact/microCompact.ts#L36-L50)
  (clears tool-result content, leaves a placeholder — same idea, client-side); compaction ≈
  [services/compact/compact.ts](cc-code/restored-src/src/services/compact/compact.ts);
  memory tool ≈ [memdir/](cc-code/restored-src/src/memdir/) + `startRelevantMemoryPrefetch`.
  Beta headers are real: [constants/betas.ts:13-14](cc-code/restored-src/src/constants/betas.ts#L13-L14).
- **Slides 4, 5 (prefill/decode, KV cache)** — GPU-level facts, correctly not in this repo.
- **Slides 19, 20, 21 (build-vs-buy, field, meta-harness)** — external. The one in-repo
  hook: this restored tree *is* the Tier-A product harness, and the Tier-B SDK surface is
  [entrypoints/sdk/](cc-code/restored-src/src/entrypoints/sdk/)
  + [QueryEngine.ts:209](cc-code/restored-src/src/QueryEngine.ts#L209).
  "They stream structured events" ← `SDKMessage` in coreTypes.ts.
- **Slides 1, 2, 8** — title / framing / contents.

---

# Three things the code says that the deck currently understates

1. **Slide 16 lists 6 hook events; the code has 20+.**
   [coreTypes.ts:25-45](cc-code/restored-src/src/entrypoints/sdk/coreTypes.ts#L25-L45) adds
   `PostToolUseFailure`, `SubagentStart`, `SubagentStop`, `PostCompact`,
   `PermissionRequest`, `PermissionDenied`, `TaskCreated`, `TaskCompleted`, `Elicitation`,
   `Setup`, `TeammateIdle`. For a bank audience, `PermissionRequest` / `PermissionDenied` /
   `PostToolUseFailure` are exactly the compliance attachment points — worth adding, it
   strengthens the "compliance lives here, not in a prompt" line.

2. **Slide 6's "tool definitions are deploy artefacts" has a stronger form in the code.**
   [tools.ts:354-357](cc-code/restored-src/src/tools.ts#L354-L357) shows built-ins are
   sorted as a *contiguous prefix* so MCP tools can't interleave. That's a concrete
   architectural move, not just a warning — it may land better than the abstract version.

3. **Slide 15's L1 claim is provable on one line.**
   [Tool.ts:750](cc-code/restored-src/src/Tool.ts#L750) — `isConcurrencySafe → false
   (assume not safe)`, right next to `isReadOnly → false` and `isDestructive → false`,
   under a comment reading *"Defaults (fail-closed where it matters)"*. A screenshot of
   those four lines would make the point faster than the sentence.
