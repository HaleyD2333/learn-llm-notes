# Slide → Code Reference Map

## Slide 3 · Vocabulary — the table is a map of the tree

The row that matters most is **Tool**: one object answers five separate questions, which is
why the loop can treat every capability identically.

```ts
// Tool.ts:362 — the interface every capability implements
export type Tool<Input extends AnyObject = AnyObject, Output = unknown, …> = {
  /**
   * One-line capability phrase used by ToolSearch for keyword matching.
   * Helps the model find this tool via keyword search when it's deferred.
   * 3–10 words, no trailing period.
   */
  searchHint?: string
  call(args, context): AsyncGenerator          // streams progress, then result
  inputSchema: Input                           // Zod → API schema + validation + UI
  isConcurrencySafe(input): boolean            // → parallel scheduling
  isEnabled(): boolean
  isReadOnly(input): boolean                   // → permission flow
  isDestructive?(input): boolean
  interruptBehavior?(): 'cancel' | 'block'     // → steering
  checkPermissions(...): Promise<PermissionResult>
}
```

| Row | Code |
|---|---|
| Model | [claude.ts:752](src/services/api/claude.ts#L752) `queryModelWithStreaming` |
| Harness | the whole `src/` tree |
| Agent | [query.ts:241](src/query.ts#L241) `queryLoop` |
| Context — "rebuilt every turn" | [query.ts:365-467](src/query.ts#L365-L467) |
| Tool | [Tool.ts:362](src/Tool.ts#L362) interface, [`buildTool`:783](src/Tool.ts#L783) |
| MCP server | [services/mcp/](src/services/mcp/), `normalization.ts` (`mcp__server__tool` prefixing) |
| Skill | [skills/loadSkillsDir.ts](src/skills/loadSkillsDir.ts), [tools/SkillTool/](src/tools/SkillTool/) |
| Subagent | [runAgent.ts:248](src/tools/AgentTool/runAgent.ts#L248) |
| Task | [tasks/types.ts](src/tasks/types.ts), [utils/task/framework.ts](src/utils/task/framework.ts) |
| Hook | [`HOOK_EVENTS` coreTypes.ts:25](src/entrypoints/sdk/coreTypes.ts#L25) |

---

## Slide 6 · Prompt cache — the strongest theory→code link in the deck

**"Exact prefix match, tools → system → messages"** — this comment *is* the slide:

```ts
// tools.ts:354 — assembleToolPool()
  // Sort each partition for prompt-cache stability, keeping built-ins as a
  // contiguous prefix. The server's claude_code_system_cache_policy places a
  // global cache breakpoint after the last prefix-matched built-in tool; a flat
  // sort would interleave MCP tools into built-ins and invalidate all downstream
  // cache keys whenever an MCP tool sorts between existing built-ins. uniqBy
  // preserves insertion order, so built-ins win on name conflict.
  // Avoid Array.toSorted (Node 20+) — we support Node 18. builtInTools is
  // readonly so copy-then-sort; allowedMcpTools is a fresh .filter() result.
  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
```

**"4 breakpoints"**:

```ts
// claude.ts:3063
export function addCacheBreakpoints(…)
  // :3078
  // Exactly one message-level cache_control marker per request.

// claude.ts:603 / :615 / :648 — the markers themselves
cache_control: getCacheControl({ querySource })

// claude.ts:404 — TTL is not static
  // mid-session overage flips from changing the cache_control TTL, which …
```

**"Volatile content goes last"** — the memoisation is a *cache-prefix* decision:

```ts
// context.ts
export const getGitStatus     = memoize(async (): Promise<string | null> => { … })  // :36
export const getSystemContext = memoize(…)                                          // :116
export const getUserContext   = memoize(…)                                          // :155
```

| Also on this slide | Code |
|---|---|
| Skip the cache write | [query.ts:696](src/query.ts#L696) `skipCacheWrite` |
| Cache-break detection | [promptCacheBreakDetection.ts](src/services/api/promptCacheBreakDetection.ts) `notifyCompaction` |
| Usage fields | [query.ts:489-491](src/query.ts#L489-L491) |

---

## Slide 7 · The loop

**"Exit is typed, not thrown"** — the outer wrapper documents the generator semantics:

```ts
// query.ts:219
export async function* query(params: QueryParams): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage,
  Terminal                                        // ← typed return, not a throw
> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  // Only reached if queryLoop returned normally. Skipped on throw (error
  // propagates through yield*) and on .return() (Return completion closes
  // both generators).
  for (const uuid of consumedCommandUuids) notifyCommandLifecycle(uuid, 'completed')
  return terminal
}
```

**"A state machine, not recursion"** — everything carried between iterations:

```ts
// query.ts:204
// Mutable state carried between loop iterations
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  // Why the previous iteration continued. Undefined on first iteration.
  // Lets tests assert recovery paths fired without inspecting message contents.
  transition: Continue | undefined
}
```

```ts
// query.ts:307 — the loop, and its only two exits
while (true) {
  // …shape context → call model → stream → tools…
  state = next; continue            // 7 continue-sites
  return { reason: 'completed' }    // 10 Terminal reasons
}
```

**The ten terminal reasons**, with their return sites:

| Reason | Line | Reason | Line |
|---|---|---|---|
| `blocking_limit` | [646](src/query.ts#L646) | `prompt_too_long` | [1175](src/query.ts#L1175) · [1182](src/query.ts#L1182) |
| `image_error` | [977](src/query.ts#L977) | `completed` | [1264](src/query.ts#L1264) · [1357](src/query.ts#L1357) |
| `model_error` | [996](src/query.ts#L996) | `stop_hook_prevented` | [1279](src/query.ts#L1279) |
| `aborted_streaming` | [1051](src/query.ts#L1051) | `aborted_tools` | [1515](src/query.ts#L1515) |
| `hook_stopped` | [1520](src/query.ts#L1520) | `max_turns` | [1711](src/query.ts#L1711) |

**"UI-agnostic, I/O injected"** ← [`ToolUseContext` Tool.ts:158](src/Tool.ts#L158)
+ [query/deps.ts](src/query/deps.ts) `productionDeps`
+ [query/config.ts](src/query/config.ts). One loop serves
[REPL.tsx:2793](src/screens/REPL.tsx#L2793) and
[QueryEngine.ts:209](src/QueryEngine.ts#L209).

> **Note for the tutorial snippet on this slide.** `if response.stop_reason != "tool_use":
> return` is the *necessary* condition, not the sufficient one — and query.ts explicitly
> refuses to use `stop_reason` at all. See §8 of `cc_detail.md`.

---

## Slide 9 · Context

Four sources, assembled fresh every iteration:

```ts
// query.ts:449 — system side
const fullSystemPrompt = asSystemPrompt(
  appendSystemContext(systemPrompt, systemContext))

// query.ts:659 — user side, at the call
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),   // :660
  systemPrompt: fullSystemPrompt,
  tools: toolUseContext.options.tools,
  …
}))
```

| Tier | Mechanism in code |
|---|---|
| Turn-local | [query.ts:365](src/query.ts#L365) — the messages array |
| Session | [sessionStorage.ts:202](src/utils/sessionStorage.ts#L202) `getTranscriptPath` JSONL |
| Project | [memdir/](src/memdir/) + CLAUDE.md via `getUserContext` |
| Organisational | [loadSkillsDir.ts](src/skills/loadSkillsDir.ts) + [tools.ts](src/tools.ts) |

"No separate personality config" ← [utils/systemPromptType.ts](src/utils/systemPromptType.ts)
`asSystemPrompt` — sections concatenated at runtime.

---

## Slide 11 · Compaction — near-perfect 1:1

The five stages are literally consecutive in the source:

```ts
// query.ts:379-467 — top of every iteration, cheapest first
messagesForQuery = await applyToolResultBudget(…)          // 1  :379
const snipResult = snipModule!.snipCompactIfNeeded(…)      // 2  :403
const microcompactResult = await deps.microcompact(…)      // 3  :414
const collapseResult =                                     // 4  :441
  await contextCollapse.applyCollapsesIfNeeded(…)
const { compactionResult, consecutiveFailures } =          // 5  :454
  await deps.autocompact(…, tracking, snipTokensFreed)     // ← only model call
```

**"Cheapest first, heavier becomes a no-op"** is stated in the source:

```ts
// query.ts:428
// Project the collapsed context view and maybe commit more collapses.
// Runs BEFORE autocompact so that if collapse gets us under the
// autocompact threshold, autocompact is a no-op and we keep granular
// context instead of a single summary.
//
// Nothing is yielded — the collapsed view is a read-time projection
// over the REPL's full history. Summary messages live in the collapse
// store, not the REPL array.
```

**"Microcompact blanks CONTENT but keeps structure"**:

```ts
// microCompact.ts:36
export const TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'

// microCompact.ts:41 — only these tools
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME, ...SHELL_TOOL_NAMES, GREP_TOOL_NAME, GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME, WEB_FETCH_TOOL_NAME, FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME,
])
```

**The circuit breaker, with its evidence and its thresholds**:

```ts
// autoCompact.ts:67
// Stop trying autocompact after this many consecutive failures.
// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

// autoCompact.ts:257 — "it does not back off, it stops"
if (tracking?.consecutiveFailures !== undefined &&
    tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
  return { wasCompacted: false }
}

// autoCompact.ts:30 / :62 — the thresholds, if asked
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000   // p99.99 of compact summary output
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
```

**Proactive vs reactive**: proactive at [query.ts:453](src/query.ts#L453),
reactive (after a real 413) at [query.ts:1119-1166](src/query.ts#L1119-L1166).

---

## Slide 12 · Exception handling

**"Errors become values"** — the catch block that makes the rule true:

```ts
// services/tools/toolExecution.ts:469
} catch (error) {
  logError(error)
  const errorMessage = error instanceof Error ? error.message : String(error)
  const toolInfo = tool ? ` (${tool.name})` : ''
  const detailedError = `Error calling tool${toolInfo}: ${errorMessage}`

  yield {
    message: createUserMessage({
      content: [{
        type: 'tool_result',
        content: `<tool_use_error>${detailedError}</tool_use_error>`,
        is_error: true,
        tool_use_id: toolUse.id,
      }],
      toolUseResult: detailedError,
      sourceToolAssistantUUID: assistantMessage.uuid,
    }),
  }
}
```

It **yields** rather than throws — the result rejoins the normal stream and the `tool_use`
block gets its pair.

**"An orphan tool_use makes the NEXT request fail"** — the generator that prevents it:

```ts
// query.ts:123
function* yieldMissingToolResultBlocks(
  assistantMessages: AssistantMessage[],
  errorMessage: string,
) {
  for (const assistantMessage of assistantMessages) {
    const toolUseBlocks = assistantMessage.message.content.filter(
      content => content.type === 'tool_use',
    ) as ToolUseBlock[]

    // Emit an interruption message for each tool use
    for (const toolUse of toolUseBlocks) {
      yield createUserMessage({
        content: [{ type: 'tool_result', content: errorMessage,
                    is_error: true, tool_use_id: toolUse.id }],
        toolUseResult: errorMessage,
        sourceToolAssistantUUID: assistantMessage.uuid,
      })
    }
  }
}
```

Called on **fallback** ([:900](src/query.ts#L900)), **error**
([:984](src/query.ts#L984)) and **abort**
([:1025](src/query.ts#L1025)).

**The withhold ladder** — why recoverable errors are kept internal:

```ts
// query.ts:166
/**
 * Is this a max_output_tokens error message? If so, the streaming loop should
 * withhold it from SDK callers until we know whether the recovery loop can
 * continue. Yielding early leaks an intermediate error to SDK callers (e.g.
 * cowork/desktop) that terminate the session on any `error` field — the
 * recovery loop keeps running but nobody is listening.
 */

// query.ts:799 — the withhold decision, inside the stream loop
let withheld = false
if (feature('CONTEXT_COLLAPSE')) {
  if (contextCollapse?.isWithheldPromptTooLong(message, isPromptTooLongMessage, querySource))
    withheld = true
}
if (reactiveCompact?.isWithheldPromptTooLong(message)) withheld = true
if (mediaRecoveryEnabled && reactiveCompact?.isWithheldMediaSizeError(message)) withheld = true
if (isWithheldMaxOutputTokens(message)) withheld = true
if (!withheld) yield yieldMessage
```

**The exhaustion rule** — the slide quotes this verbatim:

```ts
// query.ts:1168
// No recovery — surface the withheld error and exit. Do NOT fall
// through to stop hooks: the model never produced a valid response,
// so hooks have nothing meaningful to evaluate. Running stop hooks
// on prompt-too-long creates a death spiral: error → hook blocking
// → retry → error → … (the hook injects more tokens each cycle).
yield lastMessage
void executeStopFailureHooks(lastMessage, toolUseContext)
return { reason: isWithheldMedia ? 'image_error' : 'prompt_too_long' }
```

| Also on this slide | Code |
|---|---|
| Synthetic sibling errors | [StreamingToolExecutor.ts:153-205](src/services/tools/StreamingToolExecutor.ts#L153-L205) |
| Drain must run on abort | [query.ts:1015-1023](src/query.ts#L1015-L1023) |
| Bounded at 3 | [`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT` query.ts:164](src/query.ts#L164) |
| 3-10× gateway load | [withRetry.ts:58-61](src/services/api/withRetry.ts#L58-L61) |
| Bounded shutdown | [gracefulShutdown.ts:414-419](src/utils/gracefulShutdown.ts#L414-L419) |

---

## Slide 13 · Dead loops — one line, and the loop it stopped

**The latch reset policy.** Same field, two continue-sites, opposite policies:

```ts
// query.ts:1290 — inside the stop-hook blocking branch
  maxOutputTokensRecoveryCount: 0,
  // Preserve the reactive compact guard — if compact already ran and
  // couldn't recover from prompt-too-long, retrying after a stop-hook
  // blocking error will produce the same result. Resetting to false
  // here caused an infinite loop: compact → still too long → error →
  // stop hook blocking → compact → … burning thousands of API calls.
  hasAttemptedReactiveCompact,                 // ← PRESERVED
```

```ts
// query.ts:1721 — the normal next-turn branch, same field
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,          // ← RESET: a fresh turn earns a fresh try
```

**The transition guard** — how an iteration asks "how did I get here?":

```ts
// query.ts:1089 — single-shot collapse drain
if (
  feature('CONTEXT_COLLAPSE') &&
  contextCollapse &&
  state.transition?.reason !== 'collapse_drain_retry'   // ← no A→B→A oscillation
) {
```

**Provenance guards** — a subsystem refusing to fire on its own query source:

```ts
// autoCompact.ts:169
  // Recursion guards. session_memory and compact are forked agents that
  // would deadlock.
  if (querySource === 'session_memory' || querySource === 'compact') return false

  // marble_origami is the ctx-agent — if ITS context blows up and
  // autocompact fires, runPostCompactCleanup calls resetContextCollapse()
  // which destroys the MAIN thread's committed log (module-level state
  // shared across forks).
  if (feature('CONTEXT_COLLAPSE')) {
    if (querySource === 'marble_origami') return false
  }
```

| Cause | Code |
|---|---|
| 1 Error→recovery→same error | [query.ts:1290](src/query.ts#L1290) · [1168](src/query.ts#L1168) · [1258](src/query.ts#L1258) |
| 2 Unbounded retry | [autoCompact.ts:67-70](src/services/compact/autoCompact.ts#L67-L70) |
| 3 Self-recursion | [autoCompact.ts:169-183](src/services/compact/autoCompact.ts#L169-L183), [query.ts:600-603](src/query.ts#L600-L603) |
| 4 Missing caps | [query.ts:164](src/query.ts#L164) · [1705](src/query.ts#L1705) · [1506](src/query.ts#L1506) (abort path!) |
| 5 Reactive feedback | [REPL.tsx:2631](src/screens/REPL.tsx#L2631), `RELOAD_DEBOUNCE_MS=300`, [attachments.ts:1056](src/utils/attachments.ts#L1056) |
| 6 Real deadlock | [policyLimits/index.ts:68](src/services/policyLimits/index.ts#L68), [skillChangeDetector.ts:52-60](src/utils/skills/skillChangeDetector.ts#L52-L60) |

---

## Slide 14 · Concurrency — there are no threads

Verified by grep: **zero `worker_threads`, zero `new Worker()`** in the tree.

**The abort tree, and why it holds its children weakly**:

```ts
// utils/abortController.ts
/**
 * Creates a child AbortController that aborts when its parent aborts.
 * Aborting the child does NOT affect the parent.
 *
 * Memory-safe: Uses WeakRef so the parent doesn't retain abandoned children.
 * If the child is dropped without being aborted, it can still be GC'd.
 * When the child IS aborted, the parent listener is removed to prevent
 * accumulation of dead handlers.
 */
export function createChildAbortController(parent, maxListeners?): AbortController {
  const child = createAbortController(maxListeners)
  if (parent.signal.aborted) { child.abort(parent.signal.reason); return child }

  const weakChild  = new WeakRef(child)
  const weakParent = new WeakRef(parent)
  const handler = propagateAbort.bind(weakParent, weakChild)
  parent.signal.addEventListener('abort', handler, { once: true })

  child.signal.addEventListener('abort',
    removeAbortHandler.bind(weakParent, new WeakRef(handler)), { once: true })
  return child
}
```

**The abort reason is a typed protocol, not a string**:

```ts
// StreamingToolExecutor.ts:210
private getAbortReason(tool): 'sibling_error' | 'user_interrupted' | 'streaming_fallback' | null {
  if (this.discarded) return 'streaming_fallback'
  if (this.hasErrored)  return 'sibling_error'
  if (this.toolUseContext.abortController.signal.aborted) {
    // 'interrupt' means the user typed a new message while tools were
    // running. Only cancel tools whose interruptBehavior is 'cancel';
    // 'block' tools shouldn't reach here (abort isn't fired).
    if (this.toolUseContext.abortController.signal.reason === 'interrupt') {
      return this.getToolInterruptBehavior(tool) === 'cancel' ? 'user_interrupted' : null
    }
    return 'user_interrupted'
  }
  return null
}
```

**Only Bash errors cancel siblings** — the dependency-chain argument:

```ts
// StreamingToolExecutor.ts:354
if (isErrorResult) {
  thisToolErrored = true
  // Only Bash errors cancel siblings. Bash commands often have implicit
  // dependency chains (e.g. mkdir fails → subsequent commands pointless).
  // Read/WebFetch/etc are independent — one failure shouldn't nuke the rest.
  if (tool.block.name === BASH_TOOL_NAME) {
    this.hasErrored = true
    this.siblingAbortController.abort('sibling_error')
  }
}
```

**The multi-process half — async lock, never sync**:

```ts
// utils/teammateMailbox.ts:31
// Lock options: retry with backoff so concurrent callers (multiple Claudes
// in a swarm) wait for the lock instead of failing immediately. The sync
// lockSync API blocked the event loop; the async API needs explicit retries
// to achieve the same serialization semantics.
const LOCK_OPTIONS = { retries: { retries: 10, minTimeout: 5, maxTimeout: 100 } }

// :170 — and the read-modify-write fix
release = await lockfile.lock(inboxPath, { lockfilePath, ...LOCK_OPTIONS })
// Re-read messages after acquiring lock to get the latest state
const messages = await readMailbox(recipientName, teamName)
```

The one deliberate escape hatch (permission rejection must end the turn) is at
[StreamingToolExecutor.ts:304-318](src/services/tools/StreamingToolExecutor.ts#L304-L318),
citing regression #21056.

---

## Slide 15 · Multi-agent — fail-closed defaults

```ts
// Tool.ts:744 — buildTool()
 * Build a complete `Tool` from a partial definition, filling in safe defaults
 * for the commonly-stubbed methods.
 *
 * Defaults (fail-closed where it matters):
 * - `isEnabled`          → `true`
 * - `isConcurrencySafe`  → `false` (assume not safe)
 * - `isReadOnly`         → `false` (assume writes)
 * - `isDestructive`      → `false`
 * - `checkPermissions`   → `{ behavior: 'allow', updatedInput }`
```

**L1 — the entire parallelism policy is three clauses**:

```ts
// StreamingToolExecutor.ts:129
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}

// :140 — order must hold for unsafe tools
private async processQueue(): Promise<void> {
  for (const tool of this.tools) {
    if (tool.status !== 'queued') continue
    if (this.canExecuteTool(tool.isConcurrencySafe)) {
      await this.executeTool(tool)
    } else {
      // Can't execute this tool yet, and since we need to maintain order
      // for non-concurrent tools, stop here
      if (!tool.isConcurrencySafe) break        // break, not continue
    }
  }
}
```

**Ordered output despite parallel execution**:

```ts
// StreamingToolExecutor.ts:424 — a fast tool at position 3 waits for 1 and 2
if (tool.status === 'completed' && tool.results) {
  tool.status = 'yielded'
  for (const message of tool.results) yield { message, newContext: this.toolUseContext }
  markToolUseAsComplete(this.toolUseContext, tool.id)
} else if (tool.status === 'executing' && !tool.isConcurrencySafe) {
  break                                          // can't yield past a hole
}
```

**L2 — subagents are themselves concurrency-safe, so N fan out in parallel**:

```ts
// AgentTool.tsx:1273
isConcurrencySafe() {
  return true
},
```

**L3 — the promotion race** (register foreground, offer background at 2s):

```ts
// AgentTool.tsx:808
// Register as foreground task immediately so it can be backgrounded at any time
// Create the background race promise once outside the loop — otherwise
// each iteration adds a new .then() reaction to the same pending
// promise, accumulating callbacks for the lifetime of the agent.
let backgroundPromise: Promise<{ type: 'background' }> | undefined
…
backgroundPromise = registration.backgroundSignal.then(() => ({ type: 'background' as const }))

// AgentTool.tsx:63
const PROGRESS_THRESHOLD_MS = 2000;   // Show background hint after 2 seconds

// AgentTool.tsx:694 — the one place the abort tree is deliberately cut
// Don't link to parent's abort controller -- background agents should …
```

**Chain id + depth on every event**:

```ts
// query.ts:347
const queryTracking = toolUseContext.queryTracking
  ? { chainId: toolUseContext.queryTracking.chainId,
      depth:   toolUseContext.queryTracking.depth + 1 }
  : { chainId: deps.uuid(), depth: 0 }
```

L4 peers: [SendMessageTool.ts](src/tools/SendMessageTool/SendMessageTool.ts)
— broadcast `@team` (:260), request/response (:114), negotiated shutdown (:276).

---

## Slide 16 · Governance — where compliance actually attaches

```ts
// utils/permissions/permissions.ts:473
export const hasPermissionsToUseTool: CanUseToolFn = async (
  tool, input, context, assistantMessage, toolUseID,
): Promise<PermissionDecision> => {
  const result = await hasPermissionsToUseToolInner(tool, input, context)
  …
}

// precedence — deny ▸ ask ▸ allow
getDenyRules(context)    // :213
getAskRules(context)     // :223
getAllowRules(context)   // :122
```

**Modes transform the outcome globally**:

```ts
// utils/permissions/PermissionMode.ts:42
const PERMISSION_MODE_CONFIG: Partial<Record<PermissionMode, PermissionModeConfig>> = {
  default:           { title: 'Default',             external: 'default' },
  plan:              { title: 'Plan Mode',           external: 'plan' },
  acceptEdits:       { title: 'Accept edits',        external: 'acceptEdits' },
  bypassPermissions: { title: 'Bypass Permissions',  external: 'bypassPermissions', color: 'error' },
  dontAsk:           { title: "Don't Ask",           … },
}
```

**The deck lists six hook events. The source defines twenty-seven**:

```ts
// entrypoints/sdk/coreTypes.ts:25
export const HOOK_EVENTS = [
  'PreToolUse', 'PostToolUse', 'PostToolUseFailure', 'Notification',
  'UserPromptSubmit', 'SessionStart', 'SessionEnd',
  'Stop', 'StopFailure', 'SubagentStart', 'SubagentStop',
  'PreCompact', 'PostCompact',
  'PermissionRequest', 'PermissionDenied',        // ← the audit story
  'Setup', 'TeammateIdle', 'TaskCreated', 'TaskCompleted',
  'Elicitation', 'ElicitationResult', 'ConfigChange',
  'WorktreeCreate', 'WorktreeRemove',
  'InstructionsLoaded', 'CwdChanged', 'FileChanged',
] as const
```

Only some are **blocking**: `PreToolUse`, `UserPromptSubmit`, `Stop` and `PermissionRequest`
can change control flow. Wired into the loop at
[query.ts:1001](src/query.ts#L1001) (post-sampling),
[:1174](src/query.ts#L1174) (stop-failure),
[:1267](src/query.ts#L1267) (stop hooks).

Batch "ask" policy ← [yoloClassifier.ts:100](src/utils/permissions/yoloClassifier.ts#L100)
`getDefaultExternalAutoModeRules`. Read ≠ write ≠ destroy ←
[Tool.ts:404-406](src/Tool.ts#L404-L406).

---

## Slide 17 · Skills explosion — description text is rent

```ts
// tools/SkillTool/prompt.ts:22
// Skill listing gets 1% of the context window (in characters)
export const SKILL_BUDGET_CONTEXT_PERCENT = 0.01
export const CHARS_PER_TOKEN = 4
export const DEFAULT_CHAR_BUDGET = 8_000 // Fallback: 1% of 200k × 4

// Per-entry hard cap. The listing is for discovery only — the Skill tool loads
// full content on invoke, so verbose whenToUse strings waste turn-1 cache_creation
// tokens without improving match rate. Applies to all entries, including bundled,
// since the cap is generous enough to preserve the core use case.
export const MAX_LISTING_DESC_CHARS = 250
```

**Defer by measured cost, not tool count**:

```ts
// utils/toolSearch.ts:44
/**
 * Default percentage of context window at which to auto-enable tool search.
 * When MCP tool descriptions exceed this percentage (in tokens), tool search is enabled.
 */
const DEFAULT_AUTO_TOOL_SEARCH_PERCENTAGE = 10 // 10%

// Tool.ts:373 — what the model sees while a tool is deferred
/**
 * One-line capability phrase used by ToolSearch for keyword matching.
 * 3–10 words, no trailing period.
 */
searchHint?: string
```

**The `paths:` block on the slide is real frontmatter** — here is the machinery:

```ts
// skills/loadSkillsDir.ts:771 — conditional skills never enter the listing
const unconditionalSkills: Command[] = []
const newConditionalSkills: Command[] = []
for (const skill of deduplicatedSkills) {
  if (skill.type === 'prompt' && skill.paths && skill.paths.length > 0 &&
      !activatedConditionalSkillNames.has(skill.name)) {
    newConditionalSkills.push(skill)      // held back — invisible to the model
  } else {
    unconditionalSkills.push(skill)
  }
}
```

```ts
// skills/loadSkillsDir.ts:997 — activation, on a matching file operation
export function activateConditionalSkillsForPaths(filePaths: string[], cwd: string): string[] {
  for (const [name, skill] of conditionalSkills) {
    const skillIgnore = ignore().add(skill.paths)      // gitignore semantics
    for (const filePath of filePaths) {
      const relativePath = isAbsolute(filePath) ? relative(cwd, filePath) : filePath
      if (skillIgnore.ignores(relativePath)) {
        dynamicSkills.set(name, skill)                 // now visible
        conditionalSkills.delete(name)
        activated.push(name)
        break
      }
    }
  }
}
```

**Discovery must survive compaction**, or the model re-searches for tools it already has:

```ts
// utils/toolSearch.ts:495 — extractDiscoveredToolNames()
 * Compaction replaces tool_reference-bearing messages with a summary, so it
 * snapshots the discovered set onto compactMetadata.preCompactDiscoveredTools
 * on the boundary marker; this scan reads it back. Snip instead protects the
 * tool_reference-carrying messages from removal.
```

**The 97% finding** — why discovery is signal-triggered, not per-turn:

```ts
// query.ts:323
// Skill discovery prefetch — per-iteration (uses findWritePivot guard
// that returns early on non-write iterations). Discovery runs while the
// model streams and tools execute; awaited post-tools alongside the
// memory prefetch consume. Replaces the blocking assistant_turn path
// that ran inside getAttachmentMessages (97% of those calls found
// nothing in prod).
const pendingSkillPrefetch = skillPrefetch?.startSkillDiscoveryPrefetch(…)
```

---

## Slide 18 · Long-running tasks

```ts
// utils/task/diskOutput.ts
const DEFAULT_MAX_READ_BYTES = 8 * 1024 * 1024                    // 8MB   :23
export const MAX_TASK_OUTPUT_BYTES = 5 * 1024 * 1024 * 1024       // 5GB   :30

// :117 — the in-band truncation marker
if (this.#bytesWritten > MAX_TASK_OUTPUT_BYTES) {
  … `\n[output truncated: exceeded ${MAX_TASK_OUTPUT_BYTES_DISPLAY} disk cap]\n`

// :302 — "Reads only from the byte offset, up to maxBytes — never loads the full file."
export async function getTaskOutputDelta(taskId, offset, maxBytes = DEFAULT_MAX_READ_BYTES)
```

**Exactly one owner per notification**:

```ts
// utils/task/framework.ts:199
// Completed tasks are NOT notified here — each task type handles its own
// completion notification via enqueuePendingNotification(). Generating
// attachments here would race with those per-type callbacks, causing
// dual delivery (one inline attachment + one separate API turn).
```

**Task output queues behind user input**:

```ts
// utils/messageQueueManager.ts:126
// Defaults priority to 'next' (processed before task notifications).
export function enqueue(command: QueuedCommand): void {
  commandQueue.push({ ...command, priority: command.priority ?? 'next' })
}

// :142 — Convenience wrapper that defaults priority to 'later' so user input …
export function enqueuePendingNotification(command: QueuedCommand): void {
  commandQueue.push({ ...command, priority: command.priority ?? 'later' })
}
```

**The stall watchdog** — the hang that never raises an error:

```ts
// tasks/LocalShellTask/LocalShellTask.tsx:24
const STALL_CHECK_INTERVAL_MS = 5_000;
const STALL_THRESHOLD_MS      = 45_000;
const STALL_TAIL_BYTES        = 1024;

// :46 — Output-side analog of peekForStdinData: fire a one-shot
//       notification if output stops growing and the tail looks like a prompt.
const timer = setInterval(() => {
  void stat(outputPath).then(s => {
    if (s.size > lastSize) { lastSize = s.size; lastGrowth = Date.now(); return }
    if (Date.now() - lastGrowth < STALL_THRESHOLD_MS) return
    void tailFile(outputPath, STALL_TAIL_BYTES).then(({ content }) => {
      if (cancelled) return
      if (!looksLikePrompt(content)) {
        // Not a prompt — keep watching. Reset so the next check is
        // 45s out instead of re-reading the tail on every tick.
        lastGrowth = Date.now(); return
      }
      cancelled = true; clearInterval(timer)
      enqueuePendingNotification({ value: message, mode: 'task-notification',
                                   priority: 'next', agentId })
    }, () => {})
  }, () => {})
}, STALL_CHECK_INTERVAL_MS)
timer.unref()      // never keep the process alive
```

**The remediation is actionable, not just "stuck"**:

```
The command is likely blocked on an interactive prompt. Kill this task and
re-run with piped input (e.g. `echo y | command`) or a non-interactive flag
if one exists.
```

**The absence of a `<status>` tag is load-bearing**:

```ts
// LocalShellTask.tsx:79
// No <status> tag — print.ts treats <status> as a terminal
// signal and an unknown value falls through to 'completed',
// falsely closing the task for SDK consumers. Statusless
// notifications are skipped by the SDK emitter (progress ping).
```

Idempotent completion ← [LocalShellTask.tsx:105-120](src/tasks/LocalShellTask/LocalShellTask.tsx#L105-L120).
Task tools ← `TaskCreate/List/Get/Output/Update/Stop`;
[`TaskOutputTool.isConcurrencySafe → true`:160](src/tools/TaskOutputTool/TaskOutputTool.tsx#L160).

---
