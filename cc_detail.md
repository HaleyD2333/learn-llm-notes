
# The Query Loop

## The big picture

[queryLoop](cc-code/restored-src/src/query.ts#L241) is **the core agentic loop of Claude Code** — the engine that drives one user turn to completion. It's an `async function*` (async generator), so it *streams* everything out — assistant text, tool calls, tool results, errors, boundary markers — as they happen, while a caller consumes them with `for await`.

Its public wrapper is [query()](cc-code/restored-src/src/query.ts#L219), which just delegates via `yield* queryLoop(...)` and then marks queued commands "completed" on normal exit ([query.ts:230-238](cc-code/restored-src/src/query.ts#L230-L238)). All the real work is here.

## The shape: a state machine, not recursion

Despite comments mentioning "recursion," it's actually a `while (true)` loop ([query.ts:307](cc-code/restored-src/src/query.ts#L307)) over a single mutable [`State`](cc-code/restored-src/src/query.ts#L204-L217) object. Every path that wants another iteration builds a fresh `next: State = {...}`, assigns `state = next`, and `continue`s. Every terminal path does `return { reason: ... }`. So you can read the whole function as: **"call the model, react to what came back, decide whether to loop or stop."**

The `transition` field on each State records *why* the last iteration continued (`next_turn`, `reactive_compact_retry`, `max_output_tokens_recovery`, etc.) — mostly so tests can assert which recovery path fired.

## What one iteration does

**1. Prep the context** ([query.ts:365-467](cc-code/restored-src/src/query.ts#L365-L467)) — take the messages and shrink them to fit the window, in a deliberate order: tool-result budget → snip → microcompact → context-collapse → autocompact. Each is a different strategy for freeing tokens; they run before the API call so the request fits.

**2. Call the model and stream** ([query.ts:659-863](cc-code/restored-src/src/query.ts#L659-L863)) — `deps.callModel(...)` yields messages as they stream. For each one it:
- clones assistant messages to backfill tool-input display fields (without mutating the cache-bound original),
- **withholds** recoverable errors (prompt-too-long, max-output-tokens, oversized media) instead of yielding them — because it may be able to recover, and leaking an intermediate error would make SDK callers kill the session,
- collects `tool_use` blocks and, if streaming tool execution is on, kicks tools off *while the model is still streaming*.

There's a fallback-model retry wrapped around this (`attemptWithFallback`) and a big `catch` that turns real errors into synthetic assistant messages so the loop degrades gracefully.

**3. Decide what happens next.** Two cases:

- **No tool calls** (`!needsFollowUp`, [query.ts:1062](cc-code/restored-src/src/query.ts#L1062)) → the turn is trying to end. This is where all the withheld-error recovery lives: try context-collapse drain, then reactive compact, then max-output-tokens escalation/recovery. If none apply, run stop hooks (which can inject a blocking error and force another loop) and the token-budget check. Otherwise `return { reason: 'completed' }`.

- **Tool calls present** ([query.ts:1366](cc-code/restored-src/src/query.ts#L1366) onward) → execute the tools (`streamingToolExecutor` or `runTools`), yield each result, handle abort/hook-stop, gather queued-command / memory-prefetch / skill-prefetch attachments, then build the next State with `messages = [...messagesForQuery, ...assistantMessages, ...toolResults]` and loop for another model call.

## Terminal reasons

The loop only exits by `return`ing a `Terminal` — `completed`, `aborted_streaming`, `aborted_tools`, `model_error`, `prompt_too_long`, `image_error`, `blocking_limit`, `max_turns`, `stop_hook_prevented`, `hook_stopped`. That value becomes the generator's return value, which `query()` passes back up.

## Mental model

Strip away the compaction/recovery/telemetry and it's the classic agent loop:

> call model → if it asked for tools, run them and feed results back → else stop.

Everything else in these ~1500 lines is production hardening around that core: keeping the context under the token limit, recovering from four classes of recoverable API error without leaking them, honoring aborts and stop-hooks, draining queued user input mid-turn, and emitting instrumentation.

Want me to zoom in on any one subsystem — the compaction pipeline, the withhold/recovery logic, or streaming tool execution?

---
I've now read the implementations. Here's each subsystem in detail, in the order you asked.

---

# 1. Withhold / recovery logic

## The problem it solves

When you call the model, the streaming loop yields every message straight out to the caller (`for await` in the SDK/UI). But some "messages" the model layer produces are actually **recoverable errors** — the request was too big, or the output got cut off. If the loop naively yielded those, two bad things happen:

1. SDK consumers (cowork, desktop) terminate the session on *any* `error` field. So an intermediate error the loop was about to recover from would kill the session out from under the recovery code — "the recovery loop keeps running but nobody is listening" ([query.ts:166-172](cc-code/restored-src/src/query.ts#L166-L172)).
2. The user would see a scary error that then silently gets superseded.

So the loop **withholds** these errors: keeps them internally but doesn't yield them, attempts recovery, and only surfaces the error if recovery is exhausted.

## Three classes of withheld error

Inside the streaming loop ([query.ts:788-825](cc-code/restored-src/src/query.ts#L788-L825)), each streamed message is checked against several predicates. If any says "withhold," the message is **still pushed to `assistantMessages`** (so the recovery code below can find it) but **not yielded**:

```ts
let withheld = false
if (feature('CONTEXT_COLLAPSE')) {
  if (contextCollapse?.isWithheldPromptTooLong(message, ...)) withheld = true
}
if (reactiveCompact?.isWithheldPromptTooLong(message)) withheld = true
if (mediaRecoveryEnabled && reactiveCompact?.isWithheldMediaSizeError(message)) withheld = true
if (isWithheldMaxOutputTokens(message)) withheld = true
if (!withheld) yield yieldMessage
```

The three recoverable classes:
- **Prompt-too-long (413)** — the request exceeded the context window.
- **Media-size error** — an image/PDF/too-many-images payload the API rejected.
- **max-output-tokens** — the model hit its output cap mid-response ([isWithheldMaxOutputTokens](cc-code/restored-src/src/query.ts#L175-L179)).

Note the design choice at [query.ts:788-794](cc-code/restored-src/src/query.ts#L788-L794): **two independent subsystems** (context-collapse and reactiveCompact) each get a vote on withholding, and "either subsystem's withhold is sufficient." That's so turning one feature off doesn't break the other's recovery. There's a matching hoisted gate `mediaRecoveryEnabled` ([query.ts:626-627](cc-code/restored-src/src/query.ts#L626-L627)) whose whole purpose is that *withholding and recovery must agree* — the feature flag it reads (`CACHED_MAY_BE_STALE`) can flip during the 5-30s stream, and if withhold fired but recovery didn't, the message would be eaten forever. So it's read once and reused for both decisions.

## Where recovery happens

Recovery only runs when the turn is trying to **end** — i.e. `!needsFollowUp` (no tool calls, [query.ts:1062](cc-code/restored-src/src/query.ts#L1062)). At that point it looks at `lastMessage` and classifies it:

### (a) Prompt-too-long — a two-stage ladder

`isWithheld413` ([query.ts:1070-1073](cc-code/restored-src/src/query.ts#L1070-L1073)). Recovery tries the **cheap** option first, then the **expensive** one:

1. **Context-collapse drain** ([query.ts:1089-1117](cc-code/restored-src/src/query.ts#L1089-L1117)) — drains all *staged* context collapses (`recoverFromOverflow`). This keeps granular context rather than replacing it with a summary. Guarded so it fires only once: if the previous `transition` was already `collapse_drain_retry` and the retry *still* 413'd, skip straight to stage 2. On success it builds a new State with `transition: { reason: 'collapse_drain_retry' }` and `continue`s the loop.
2. **Reactive compact** ([query.ts:1119-1166](cc-code/restored-src/src/query.ts#L1119-L1166)) — `tryReactiveCompact` produces a full summary. Guarded by `hasAttemptedReactiveCompact` so it can't spiral. On success: build post-compact messages, yield them, and `continue` with `hasAttemptedReactiveCompact: true`.

### (b) Media-size — `isWithheldMedia` ([query.ts:1082-1084](cc-code/restored-src/src/query.ts#L1082-L1084))

Recoverable via reactive compact's strip-retry (it strips images). It **skips the collapse drain** (collapse doesn't strip images) and goes straight to `tryReactiveCompact`. Shares the same code path as (a) at [query.ts:1119](cc-code/restored-src/src/query.ts#L1119).

### (c) max-output-tokens — escalate, then multi-turn nudge

[query.ts:1188-1256](cc-code/restored-src/src/query.ts#L1188-L1256), two-tier:

1. **Escalation** ([query.ts:1199-1221](cc-code/restored-src/src/query.ts#L1199-L1221)) — if the request used the capped 8k default, retry the *same* request once at 64k (`ESCALATED_MAX_TOKENS`) via `maxOutputTokensOverride`. No meta-message, no dance.
2. **Multi-turn recovery** ([query.ts:1223-1252](cc-code/restored-src/src/query.ts#L1223-L1252)) — if 64k also capped, inject a meta user message ("Output token limit hit. Resume directly — no apology...") and loop. Bounded by `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3` ([query.ts:164](cc-code/restored-src/src/query.ts#L164)).

## The exhaustion rule

Every recovery path shares one crucial rule: **if recovery fails, surface the withheld error and return — do NOT fall through to stop hooks** ([query.ts:1168-1183](cc-code/restored-src/src/query.ts#L1168-L1183)):

```ts
// No recovery — surface the withheld error and exit. Do NOT fall
// through to stop hooks ... Running stop hooks on prompt-too-long
// creates a death spiral: error → hook blocking → retry → error → …
yield lastMessage
void executeStopFailureHooks(lastMessage, toolUseContext)
return { reason: isWithheldMedia ? 'image_error' : 'prompt_too_long' }
```

The whole pattern is: **withhold → attempt cheapest recovery → escalate → on exhaustion, finally yield the real error and terminate.** The `transition` field records which rung of the ladder fired, mostly so tests can assert the path without inspecting message contents.

---

# 2. Streaming tool execution

Source: [StreamingToolExecutor.ts](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts). Gated by `config.gates.streamingToolExecution` ([query.ts:561](cc-code/restored-src/src/query.ts#L561)); the fallback is the simpler `runTools` ([toolOrchestration.ts](cc-code/restored-src/src/services/tools/toolOrchestration.ts)).

## The core idea

Normally you'd wait for the model to *finish* streaming, collect all its `tool_use` blocks, then run them. The `StreamingToolExecutor` instead **starts running each tool the moment its block finishes streaming** — while the model is still emitting later blocks. On a multi-tool turn that overlaps tool latency with model latency.

Look at the wiring in the stream loop: as each assistant message arrives, `streamingToolExecutor.addTool(toolBlock, message)` is called ([query.ts:841-843](cc-code/restored-src/src/query.ts#L841-L843)), and immediately after, completed results are drained non-blockingly with `getCompletedResults()` ([query.ts:851](cc-code/restored-src/src/query.ts#L851)) — all *inside* the `for await` over the model stream.

## Concurrency model

Each tool is a [`TrackedTool`](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L21-L32) with a status: `queued → executing → completed → yielded`.

The gate is [`canExecuteTool`](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L129-L135):

```ts
executingTools.length === 0 ||
(isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
```

- **Concurrency-safe** tools (read-only: Read, Grep, Glob, WebFetch...) run **in parallel** with other safe tools.
- **Unsafe** tools (Edit, Write, Bash — anything that mutates state) run **exclusively**: alone, nothing else executing.

`isConcurrencySafe` is decided per-tool by parsing the input against the tool's schema and calling `toolDefinition.isConcurrencySafe(input)` ([lines 104-113](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L104-L113)) — so it's input-dependent, not just tool-type-dependent.

[`processQueue`](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L140-L151) walks the queue in order and starts what it can. Crucially, when it hits an unsafe tool it *can't* start yet, it **`break`s** rather than skipping ahead — because **order must be preserved for unsafe tools**. You can't run a later read past a pending write.

## Ordered output despite parallel execution

Tools may *finish* out of order, but results must be **yielded in the order the tools were received** (the API requires `tool_result` blocks matching `tool_use` order). [`getCompletedResults`](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L412-L440) walks tools in order and:
- yields any `pendingProgress` immediately (progress messages aren't order-sensitive),
- yields a completed tool's results and marks it `yielded`,
- **stops at the first unsafe tool still `executing`** ([line 436-438](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L436-L438)) — can't yield past a hole in the order.

So a fast tool at position 3 buffers its results until tools 1 and 2 have been yielded.

## Error cascading (the sibling-abort mechanism)

This is the subtle part. There's a `siblingAbortController` that's a **child** of the query's abort controller ([lines 45-48](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L45-L48)). Aborting it kills sibling tools **without ending the turn**.

The rule ([lines 354-364](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L354-L364)): **only Bash errors cancel siblings.**

```ts
if (tool.block.name === BASH_TOOL_NAME) {
  this.hasErrored = true
  this.siblingAbortController.abort('sibling_error')
}
```

Rationale in the comment: Bash commands often have implicit dependency chains (`mkdir` fails → later commands are pointless), but Read/WebFetch are independent — one failing shouldn't nuke the rest. Cancelled siblings get a **synthetic error** `tool_result` ([createSyntheticErrorMessage](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L153-L205)) so every `tool_use` still has a matching `tool_result` (the API errors otherwise). There's careful bookkeeping (`thisToolErrored`, [line 330](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L330)) so the tool that *caused* the error doesn't also get a "sibling error" message.

There's also a per-tool `toolAbortController` ([lines 301-318](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L301-L318)) whose abort **bubbles up** to the query controller for the permission-rejection case — so an ExitPlanMode "clear context + auto" reject actually ends the turn instead of sending REJECT_MESSAGE to the model (a cited regression, #21056).

## Two drain methods

- **`getCompletedResults()`** (sync generator) — non-blocking; grabs whatever's ready *right now*. Called during streaming.
- **`getRemainingResults()`** (async generator, [lines 453-490](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L453-L490)) — called *after* streaming ([query.ts:1381](cc-code/restored-src/src/query.ts#L1381)); loops `processQueue` → drain → `Promise.race` on executing tools' promises (or progress availability) until every tool is `yielded`. This is what awaits the tools that were still running when the model finished.

## `discard()` — the fallback interaction

When a streaming model-fallback fires ([query.ts:733-740](cc-code/restored-src/src/query.ts#L733-L740)), the executor is `discard()`ed and rebuilt fresh. Otherwise orphan `tool_result`s with old `tool_use_id`s (from the abandoned attempt) would leak into the retry with the new model's response.

---

# 3. Compaction

Compaction is the umbrella for "keep the context under the token limit." It is **not one mechanism** — it's a **pipeline of five**, run in a deliberate order at the top of every iteration ([query.ts:365-467](cc-code/restored-src/src/query.ts#L365-L467)), cheapest/most-granular first so a heavier one becomes a no-op if a lighter one already got you under threshold.

## The pipeline order (and why)

```
tool-result budget → snip → microcompact → context-collapse → autocompact
```

**1. Tool-result budget** ([query.ts:379-394](cc-code/restored-src/src/query.ts#L379-L394)) — `applyToolResultBudget` caps aggregate tool-result size by replacing content by `tool_use_id`. Runs *first* and, per the comment, composes cleanly with microcompact because cached-MC operates purely by id and never inspects content.

**2. Snip** ([query.ts:400-410](cc-code/restored-src/src/query.ts#L400-L410), feature `HISTORY_SNIP`) — removes messages and reports `snipTokensFreed`. That number gets **plumbed into autocompact's threshold check** because the surviving assistant message's `usage` still reflects *pre-snip* context — `tokenCountWithEstimation` alone can't see what snip removed ([autoCompact.ts:225](cc-code/restored-src/src/services/compact/autoCompact.ts#L225)). This threading appears everywhere downstream and is a recurring correctness concern.

**3. Microcompact** ([query.ts:413-426](cc-code/restored-src/src/query.ts#L413-L426), [microCompact.ts](cc-code/restored-src/src/services/compact/microCompact.ts)) — targeted, cheap, **no model call**. It clears the *content of old tool results* for a fixed set of `COMPACTABLE_TOOLS` ([lines 41-50](cc-code/restored-src/src/services/compact/microCompact.ts#L41-L50): Read, Bash, Grep, Glob, WebSearch, WebFetch, Edit, Write), replacing them with `[Old tool result content cleared]`. The conversation *structure* stays; only stale bulky payloads go. There's an ant-only "cached microcompact" variant that edits the prompt cache directly — which is why the boundary message is *deferred* until after the API response, so it can report the real `cache_deleted_input_tokens` ([query.ts:870-892](cc-code/restored-src/src/query.ts#L870-L892)).

**4. Context-collapse** ([query.ts:440-447](cc-code/restored-src/src/query.ts#L440-L447), feature `CONTEXT_COLLAPSE`) — runs *before* autocompact so that if collapsing gets you under threshold, autocompact no-ops and you keep granular context instead of a summary. Its key property ([query.ts:428-439](cc-code/restored-src/src/query.ts#L428-L439)): **nothing is yielded**. The collapsed view is a *read-time projection* over the REPL's full history — summaries live in a separate collapse store, not the message array — and `projectView()` replays a commit log on every entry, which is what makes collapses persist across turns.

**5. Autocompact** ([query.ts:453-543](cc-code/restored-src/src/query.ts#L453-L543), [autoCompact.ts](cc-code/restored-src/src/services/compact/autoCompact.ts)) — the heavyweight last resort: a **model call** that summarizes the whole conversation into a summary that *replaces* the messages.

## How autocompact decides and acts

**Threshold math** ([autoCompact.ts:33-91](cc-code/restored-src/src/services/compact/autoCompact.ts#L33-L91)):
- `effectiveContextWindow = contextWindow − reservedForSummary` (reserve ≤20k tokens for the summary output, based on p99.99 of summary size).
- `autocompactThreshold = effectiveContextWindow − 13k` buffer.
- `shouldAutoCompact` ([lines 160-239](cc-code/restored-src/src/services/compact/autoCompact.ts#L160-L239)) returns true when `tokenCount − snipTokensFreed ≥ threshold`.

**Suppression guards** — `shouldAutoCompact` bows out in several cases:
- Forked-agent query sources (`session_memory`, `compact`, `marble_origami`) would **deadlock** — they're the very agents doing the compacting ([lines 171-183](cc-code/restored-src/src/services/compact/autoCompact.ts#L171-L183)).
- **Reactive-only mode** and **context-collapse mode** disable proactive autocompact entirely ([lines 189-223](cc-code/restored-src/src/services/compact/autoCompact.ts#L189-L223)). The collapse comment is illuminating: autocompact fires at ~93% of effective window, which sits *between* collapse's 90% commit-start and 95% blocking-spawn — so if both ran, autocompact would race collapse and usually win, nuking the granular context collapse was about to save. So collapse, when on, *owns* the headroom problem.

**Circuit breaker** ([lines 67-70, 257-265, 334-350](cc-code/restored-src/src/services/compact/autoCompact.ts#L257-L265)) — stops after `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`. The cited motivation is stark: 1,279 sessions hit 50+ consecutive failures (up to 3,272), wasting ~250K API calls/day, when context was irrecoverably over-limit. The failure count rides in `autoCompactTracking.consecutiveFailures`, threaded back through the query loop's State ([query.ts:536-543](cc-code/restored-src/src/query.ts#L536-L543)) so the next iteration can skip futile retries.

**Two implementations** ([autoCompact.ts:287-333](cc-code/restored-src/src/services/compact/autoCompact.ts#L287-L333)):
1. **Session-memory compaction** tried *first* (experiment) — prunes messages using a persisted memory store.
2. **Legacy `compactConversation`** — the full summarize-and-replace.

Either way, `runPostCompactCleanup` runs, `setLastSummarizedMessageId(undefined)` resets (old UUIDs no longer exist after replacement), and cache-break detection is notified so the post-compact token drop isn't flagged as a spurious prompt-cache break.

## Back in the loop after a compact

When `compactionResult` comes back ([query.ts:470-535](cc-code/restored-src/src/query.ts#L470-L535)): log a rich `tengu_auto_compact_succeeded` event, reset `tracking` to reflect the newest compact, build `postCompactMessages` via `buildPostCompactMessages`, **yield each** so the UI/transcript sees the boundary, and set `messagesForQuery = postCompactMessages` — the *same* iteration then proceeds to the API call with the shrunk context. There's also `task_budget` accounting ([query.ts:508-515](cc-code/restored-src/src/query.ts#L508-L515)): it captures the pre-compact final context window so the server, which after a compact only sees the summary, doesn't under-count spend.

## Proactive vs reactive — the two moments compaction can fire

- **Proactive** = this pipeline, *before* the API call, driven by an estimated token count crossing a threshold.
- **Reactive** = *after* the API call rejects with a real 413, in the withhold/recovery section (subsystem #1). Reactive compact is the safety net for when the estimate was wrong and the server actually rejected.

That's the connective tissue between all three subsystems: **compaction** shrinks context proactively each turn; if it under-shoots, the model returns a 413 that the **withhold/recovery** logic catches and hands to *reactive* compaction; and throughout, **streaming tool execution** is what fills the context with tool results in the first place (which is exactly what tool-result-budget and microcompact then prune). `snipTokensFreed` and `taskBudgetRemaining` are the two values threaded across all of it to keep the accounting honest.

Want me to trace a concrete scenario end-to-end — e.g. a turn that overflows, 413s, reactively compacts, and retries — through all three?


# Core Questions

Answered from the perspective of someone *building* a harness, with the patterns
in `cc-code/restored-src/` as the reference implementation.

---

## 1. Exception handling + "multi-thread, multi-process, async" simultaneously

### The first design decision: there are no threads

Grep the whole codebase and you find zero `worker_threads` and zero `new Worker()`.
Concurrency comes from exactly three sources, and keeping them distinct is what makes
the error model tractable:

| Layer | Mechanism | Failure isolation |
|---|---|---|
| **Async** | Event loop + async generators | try/catch inside the generator |
| **Sub-process** | `child_process` / `Bun.spawn` (Bash, ripgrep, LSP, hooks) | Exit code + abort signal |
| **Multi-process agents** | Separate CLI processes (tmux/iTerm panes, remote) | Files + locks, no shared memory |

Takeaway: **don't reach for threads.** The workload is IO-bound (model latency,
subprocess latency), so async gets you all the concurrency; the moment you need true
parallel isolation you jump straight to *processes* — which forces serializable
communication and makes crash isolation free.

### Pattern A: errors become values at the tool boundary

The single most important rule: **a tool must never throw into the loop.**
[toolExecution.ts:469-486](cc-code/restored-src/src/services/tools/toolExecution.ts#L469-L486)
wraps every tool invocation:

```ts
} catch (error) {
  logError(error)
  const detailedError = `Error calling tool${toolInfo}: ${errorMessage}`
  yield { message: createUserMessage({
    content: [{ type: 'tool_result', content: `<tool_use_error>${detailedError}</tool_use_error>`,
      is_error: true, tool_use_id: toolUse.id }], ... }) }
}
```

The exception is converted into a `tool_result` with `is_error: true` and *yielded*.
Why: the API **requires** every `tool_use` block to have a matching `tool_result`. An
uncaught throw leaves an orphan `tool_use`, and the *next* request 400s — a crash that
manifests one turn later, far from its cause.

This invariant shows up as defensive code in three more places:

- [`yieldMissingToolResultBlocks`](cc-code/restored-src/src/query.ts#L123-L149) — a generator
  that synthesizes error results for every dangling `tool_use`, called on model-fallback
  ([query.ts:900](cc-code/restored-src/src/query.ts#L900)), on query error
  ([query.ts:984](cc-code/restored-src/src/query.ts#L984)), and on abort
  ([query.ts:1025](cc-code/restored-src/src/query.ts#L1025)).
- [`createSyntheticErrorMessage`](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L153-L205)
  — same job for cancelled siblings.
- `getRemainingResults()` **must** be consumed even on abort
  ([query.ts:1015-1023](cc-code/restored-src/src/query.ts#L1015-L1023)), precisely so the
  executor can emit those synthetics.

> **Rule:** every unit of concurrent work must have a *guaranteed terminal message*,
> produced on the happy path, the error path, and the cancellation path.

### Pattern B: async generators as the concurrency primitive

`async function*` does triple duty: streaming, backpressure, and structured cancellation.
Three properties the harness leans on:

**1. `yield*` propagates exceptions transparently.**
[query.ts:230-237](cc-code/restored-src/src/query.ts#L230-L237) documents this precisely:

```ts
const terminal = yield* queryLoop(params, consumedCommandUuids)
// Only reached if queryLoop returned normally. Skipped on throw (error
// propagates through yield*) and on .return() (Return completion closes
// both generators).
```

So `yield*` gives a `finally`-like semantic for free: post-processing after a delegated
generator runs *only* on clean completion. Used deliberately — a failed turn leaves
commands marked `started` without `completed`, producing an observable asymmetry signal.

**2. `using` for disposal on all exit paths.**
[query.ts:301-304](cc-code/restored-src/src/query.ts#L301-L304):

```ts
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(...)
```

Explicit resource management. A generator can be abandoned mid-iteration (caller breaks
the `for await`), and `using` is the only thing that reliably fires cleanup for that case.

**3. Fire-and-forget is marked, not accidental.** `void executePostSamplingHooks(...)`
([query.ts:1001](cc-code/restored-src/src/query.ts#L1001)), `void executeStopFailureHooks(...)`.
The `void` operator is a lint-enforced marker meaning "intentionally not awaited." Every
one has a `.catch()` or is internally guarded — an unhandled rejection in Node kills the
process. The codebase reasons about this explicitly: main.tsx has three separate comments
about "suppress transient unhandledRejection."

### Pattern C: hierarchical cancellation via AbortController trees

The mechanism that unifies async and subprocess cancellation.
[abortController.ts](cc-code/restored-src/src/utils/abortController.ts) builds a **tree**:

```
query.abortController
 └─ siblingAbortController      (StreamingToolExecutor, per-batch)
     └─ toolAbortController     (per-tool; Bash subprocesses listen here)
```

Three details worth stealing:

**Directionality is asymmetric by design** — `createChildAbortController`: "Aborting the
child does NOT affect the parent." So the executor can kill sibling tools without ending
the turn
([StreamingToolExecutor.ts:45-48](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L45-L48)).
But there is a deliberate **escape hatch** where a child *does* bubble up
([lines 304-318](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L304-L318)):
permission rejection must end the turn, so that specific abort re-fires on the parent.
Regression #21056 is cited as what happens when it doesn't.

**Memory safety via `WeakRef`.** Naïve parent→child listener registration leaks: a
long-lived parent accumulates dead handlers for every child that ever existed. The
implementation holds both directions weakly and self-removes the parent listener when the
child aborts. On a long session with thousands of tool calls this is the difference
between stable and OOM.

**`setMaxListeners(50)`** — dozens of concurrent listeners on one signal is normal here,
and Node's default-10 warning would be noise.

**The abort *reason* is a typed protocol, not a string:** `'interrupt'` (user typed a new
message) vs `'sibling_error'` vs everything else.
[`getAbortReason`](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts#L210-L231)
dispatches on it, and `'interrupt'` only cancels tools whose `interruptBehavior()` returns
`'cancel'` — a Write mid-flight keeps running.

### Pattern D: retry lives below the loop, not in it

[withRetry.ts](cc-code/restored-src/src/services/api/withRetry.ts) handles
429/529/ECONNRESET with backoff *inside* `deps.callModel`, so the query loop never sees a
transient network error. What it *does* see is a typed control-flow exception:
`FallbackTriggeredError`, caught at [query.ts:894](cc-code/restored-src/src/query.ts#L894)
to switch models and retry the whole request.

The retry policy is **query-source-aware**
([withRetry.ts:58-61](cc-code/restored-src/src/services/api/withRetry.ts#L58-L61)): only
calls a user is actually waiting on retry 529s. Summaries, titles, classifiers bail
immediately, because "during a capacity cascade each retry is 3-10× gateway load."

> **Layering rule:** transient/infrastructure errors are retried at the transport layer;
> semantic errors (413, max_output_tokens) become *messages* the loop can reason about;
> only programming bugs actually throw.

### Pattern E: multi-process coordination through locked files

For real multi-process work (swarm teammates in separate panes), there is no shared
memory — coordination is a **file-based mailbox**
([teammateMailbox.ts](cc-code/restored-src/src/utils/teammateMailbox.ts)) at
`.claude/teams/{team}/inboxes/{agent}.json`:

```ts
const LOCK_OPTIONS = { retries: { retries: 10, minTimeout: 5, maxTimeout: 100 } }
```

Three things to copy:

- **Async lock with retry/backoff, not sync.** The comment is explicit: "The sync
  `lockSync` API blocked the event loop; the async API needs explicit retries to achieve
  the same serialization semantics." Blocking the event loop inside a harness stalls
  streaming, UI, *and* every other tool.
- **Re-read after acquiring the lock**
  ([line 170](cc-code/restored-src/src/utils/teammateMailbox.ts#L170)) — the classic
  read-modify-write race.
- **Create-then-lock with `flag: 'wx'`** and tolerate `EEXIST`, because `proper-lockfile`
  needs the target to exist.

Note the failure mode: mailbox write errors are **logged and swallowed**, not thrown. A
failed message delivery degrades the team; a thrown exception kills the agent.

### Pattern F: bounded, failsafe shutdown

[gracefulShutdown.ts](cc-code/restored-src/src/utils/gracefulShutdown.ts) — cleanup runs
under a hard timer:

```
// Failsafe: guarantee process exits even if cleanup hangs (e.g., MCP connections).
// Budget = max(5s, hook budget + 3.5s headroom for cleanup + analytics flush).
```

Terminal modes are restored **first** (so a hung cleanup doesn't leave a broken terminal),
then session flush, then a `Promise.race` against the timeout. Cleanup functions come from
a [registry](cc-code/restored-src/src/utils/cleanupRegistry.ts) so subsystems self-register.

> **Rule:** every cleanup path needs a timeout. In a harness, "hangs on exit" is reported
> as "crashed."

---

## 2. Dead loops: causes and prevention

This codebase has been burned repeatedly, and the comments quantify it. Six distinct
classes:

### Cause 1: error → recovery → same error (the "death spiral")

The canonical one, documented three times in query.ts. From
[query.ts:1290-1297](cc-code/restored-src/src/query.ts#L1290-L1297):

> "Resetting to false here caused an infinite loop: compact → still too long → error →
> stop hook blocking → compact → … burning thousands of API calls."

And [query.ts:1168-1172](cc-code/restored-src/src/query.ts#L1168-L1172):

> "Running stop hooks on prompt-too-long creates a death spiral: error → hook blocking →
> retry → error → … (the hook injects more tokens each cycle)."

**Two defenses, both used:**

**(a) One-shot latches carried in loop state.** `hasAttemptedReactiveCompact` is a field on
[`State`](cc-code/restored-src/src/query.ts#L204-L217). The subtle part is which
continue-sites *preserve* it vs *reset* it. The stop-hook branch deliberately preserves it
([query.ts:1297](cc-code/restored-src/src/query.ts#L1297)) — that single line is the fix for
the loop above. The normal next-turn branch resets it
([query.ts:1721](cc-code/restored-src/src/query.ts#L1721)) because a fresh turn legitimately
gets a fresh attempt.

**(b) Guard on the previous transition.**
[query.ts:1089-1093](cc-code/restored-src/src/query.ts#L1089-L1093):

```ts
if (feature('CONTEXT_COLLAPSE') && contextCollapse &&
    state.transition?.reason !== 'collapse_drain_retry') {
```

The `transition` field exists so an iteration can ask *"how did I get here?"* — a cheap way
to prevent A→B→A oscillation without a full history.

> **Design rule:** every recovery path needs a latch, and you must decide *explicitly* at
> each continue-site whether the latch resets. Defaulting to reset is how you get the spiral.

### Cause 2: unbounded retry on a permanently-failing operation

The autocompact circuit breaker
([autoCompact.ts:67-70](cc-code/restored-src/src/services/compact/autoCompact.ts#L67-L70))
with real numbers:

> "BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272) in a single
> session, wasting ~250K API calls/day globally."

`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`, and the count is **threaded through loop state**
([autoCompact.ts:341-349](cc-code/restored-src/src/services/compact/autoCompact.ts#L341-L349)
→ [query.ts:536-543](cc-code/restored-src/src/query.ts#L536-L543)), not stored in a module
global — so it's per-conversation and testable.

Note it doesn't retry-with-backoff; it **stops**. When the input is irrecoverable, backoff
just makes the same failure slower.

### Cause 3: re-entrancy / self-recursion

A compaction agent whose own context overflows would trigger compaction. Guarded by
**query-source blacklists**
([autoCompact.ts:169-183](cc-code/restored-src/src/services/compact/autoCompact.ts#L169-L183)):

```ts
if (querySource === 'session_memory' || querySource === 'compact') return false
if (feature('CONTEXT_COLLAPSE')) { if (querySource === 'marble_origami') return false }
```

The `marble_origami` case is nastier than plain recursion: that agent's compaction would
call `resetContextCollapse()`, which destroys the **main thread's** committed log —
module-level state shared across forks. The same guard appears in the query loop's
blocking-limit check ([query.ts:602-603](cc-code/restored-src/src/query.ts#L602-L603)):
"these are forked agents that inherit the full conversation and would deadlock if blocked
here."

> **Rule:** tag every model call with its *provenance* (`QuerySource`) and let subsystems
> refuse to fire on their own provenance. A generic "am I recursing?" flag can't express this.

### Cause 4: hard iteration caps

`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`, `maxTurns`
([query.ts:1705-1712](cc-code/restored-src/src/query.ts#L1705-L1712)),
`DEFAULT_MAX_RETRIES = 10`, `maxDepth = 5` in error-cause walking, `1000/page × 100 pages`
in session ingress. Also token budget as a *soft* cap
([query.ts:1308-1355](cc-code/restored-src/src/query.ts#L1308-L1355)) — with a "diminishing
returns" early stop, a semantic cap rather than a counter.

The maxTurns check also runs on the **abort** path
([query.ts:1506-1514](cc-code/restored-src/src/query.ts#L1506-L1514)). Caps checked on only
the happy path aren't caps.

### Cause 5: feedback loops in reactive/event systems

Three flavors:

- **Event → error → event.**
  [REPL.tsx:2631-2633](cc-code/restored-src/src/screens/REPL.tsx#L2631-L2633): "Block ticks
  on API errors to prevent tick → error → tick runaway loops." A `contextBlocked` flag set
  on any API error, cleared on a successful response.
- **Watcher → reload → watcher.**
  [skillChangeDetector.ts](cc-code/restored-src/src/utils/skills/skillChangeDetector.ts) uses
  `RELOAD_DEBOUNCE_MS = 300`, and the comment says unbounded reloads "can **deadlock the
  event loop** when dozens of events fire in rapid succession." Plus
  `FILE_STABILITY_THRESHOLD_MS = 1000` and a 2s polling interval.
- **Zero-duration wake loops.**
  [attachments.ts:1056](cc-code/restored-src/src/utils/attachments.ts#L1056): "wake
  immediately with 0ms duration in an infinite loop" — a timer whose computed delay can be 0.

### Cause 6: real deadlock (mutual waiting)

Two sub-kinds, both present:

**Promise deadlock** — something awaits an init that may never happen. Fix is a timeout,
used identically in
[remoteManagedSettings](cc-code/restored-src/src/services/remoteManagedSettings/index.ts#L64)
and [policyLimits](cc-code/restored-src/src/services/policyLimits/index.ts#L68): "Timeout for
the loading promise to prevent deadlocks if `load*()` is never called."

**Runtime-level deadlock** —
[skillChangeDetector.ts:52-60](cc-code/restored-src/src/utils/skills/skillChangeDetector.ts#L52-L60)
documents a Bun `PathWatcherManager` bug where closing a watcher while the watcher thread
delivers events hangs **both threads in `__ulock_wait2` forever**. Workaround: stat-polling
under Bun, "No FSWatcher = no deadlock." A pointed reminder that when you *do* have real
threads, you inherit the runtime's deadlocks.

### Checklist for a harness builder

1. Make the loop a **state machine with named transitions**, not recursion — you can then
   guard on "how did I get here."
2. Every recovery path: **latch** + explicit reset policy per continue-site.
3. Every retry: **circuit breaker** with the counter in loop state.
4. Every model call: **provenance tag**, so subsystems can refuse self-recursion.
5. Every iterative construct: a **hard cap**, checked on abort paths too.
6. Every watcher/event source: **debounce** + a floor on computed delays.
7. Every `await` on external init: a **timeout**.
8. Make loop-exit **explicit and typed** — `Terminal` in query.ts is a discriminated union
   of 10 reasons. "The loop ended and I don't know why" is unobservable in production.

---

## 3. Managing skill / MCP explosion, and knowing what to trigger

This is the problem the codebase has invested the most recent engineering in, and it has
**two structurally different answers** for tools vs. skills.

### The core constraint: description text is rent

Every tool definition and skill description sits in the prompt on **every request**. 100
MCP tools ≈ tens of thousands of tokens of permanent overhead, paid per turn, competing with
actual work. The whole design is: **pay for a name, not a definition.**

### Answer A (MCP tools): server-side deferred loading

[toolSearch.ts](cc-code/restored-src/src/utils/toolSearch.ts) implements a two-tier scheme
using the API's `defer_loading` beta:

1. Tools marked deferred are sent with `defer_loading: true` — the model sees
   **name + one-line capability phrase only**
   ([Tool.ts:373](cc-code/restored-src/src/Tool.ts#L373): "One-line capability phrase used by
   ToolSearch for keyword matching").
2. The model calls `ToolSearchTool` with a query; results come back as `tool_reference`
   blocks that the API **expands server-side** into full definitions.

Four implementation details that matter:

**Auto-enable by measured cost, not tool count.** `DEFAULT_AUTO_TOOL_SEARCH_PERCENTAGE = 10`
— defer when deferred-tool definitions exceed **10% of the context window**. Exact token
count via the counting API (memoized on the deferred tool-name set), with a
`CHARS_PER_TOKEN = 2.5` heuristic fallback. Counting *tools* would be wrong; one verbose tool
can cost more than twenty terse ones.

**Discovered tools persist across the conversation** —
[`extractDiscoveredToolNames`](cc-code/restored-src/src/utils/toolSearch.ts#L495-L556) scans
message history for `tool_reference` blocks so subsequent requests include only what's been
discovered. Critically, it survives compaction: compact snapshots the set onto
`compactMetadata.preCompactDiscoveredTools`, and snip *protects* tool_reference-carrying
messages from removal. **A discovery cache that resets on compaction is worse than none** —
the model re-searches for tools it already has.

**Escape hatches for eligibility.** `modelSupportsToolReference` uses a **negative** test —
new models are assumed to work unless pattern-matched as unsupported (Haiku), with the
pattern list live-configurable via GrowthBook. Positive allowlists rot; you ship a model and
it silently loses the feature.

**Announce pool changes as a delta.**
[`getDeferredToolsDelta`](cc-code/restored-src/src/utils/toolSearch.ts#L600-L680) diffs the
current deferred pool against what's already been announced, and only sends added/removed.
MCP servers connect and disconnect mid-session; re-sending the whole roster every turn
destroys the prompt cache. Note the subtlety: a tool that stopped being deferred but is still
available is **not** reported as removed — "telling the model 'no longer available' would be
wrong."

### Answer B (skills): progressive disclosure + a hard budget

Skills use a different shape because they're markdown, not callable APIs.
[SkillTool/prompt.ts](cc-code/restored-src/src/tools/SkillTool/prompt.ts):

```ts
export const SKILL_BUDGET_CONTEXT_PERCENT = 0.01   // 1% of context
export const MAX_LISTING_DESC_CHARS = 250          // per-entry cap
```

The listing is **name + ≤250-char description**; full content loads only on invoke. The
rationale is explicitly economic: "the listing is for discovery only… verbose `whenToUse`
strings waste turn-1 `cache_creation` tokens without improving match rate."

`formatCommandsWithinBudget` degrades gracefully — full descriptions if they fit, otherwise
truncate toward `MIN_DESC_LENGTH = 20`. The budget is a hard ceiling; skills don't get to
grow the prompt without limit.

### Answer C — the conflicting-skills problem: **conditional activation by path**

Skills support a `paths:` frontmatter field, parsed at
[loadSkillsDir.ts:159-178](cc-code/restored-src/src/skills/loadSkillsDir.ts#L159-L178) with
gitignore-style patterns. At load time skills split into two pools
([lines 771-796](cc-code/restored-src/src/skills/loadSkillsDir.ts#L771-L796)):

```ts
const unconditionalSkills: Command[] = []   // always listed
const newConditionalSkills: Command[] = []  // held back, invisible
```

Conditional skills are **completely absent from the model's view** until
[`activateConditionalSkillsForPaths`](cc-code/restored-src/src/skills/loadSkillsDir.ts#L997-L1056)
fires — triggered when a file operation touches a matching path — at which point the skill
moves from `conditionalSkills` into `dynamicSkills` and appears in the listing.

**This is the direct answer to the asset-class problem.** An equities-research skill and a
credit-research skill shouldn't both be visible. Scope them:

```yaml
---
name: equity-research
description: Valuation and screening workflow for listed equities.
paths: |
  portfolios/equities/**
  research/equities/**
---
```

Once the model reads `portfolios/equities/2026Q1.md`, the equities skill activates and the
credit skill stays invisible. No prompt cost, and — more importantly — **no opportunity for
the model to pick the wrong one**, because it never sees it.

Related mechanisms in the same family:

- `dynamic_skill` attachments
  ([attachments.ts:525-527](cc-code/restored-src/src/utils/attachments.ts#L525-L527)) —
  skills discovered by entering a *directory* that contains a skills folder.
- `disable-model-invocation: true` — user-only skills, excluded from the model's listing
  entirely ([SkillTool.ts:412](cc-code/restored-src/src/tools/SkillTool/SkillTool.ts#L412)).
- Per-agent filtering:
  [`getSkillListingAttachments`](cc-code/restored-src/src/utils/attachments.ts#L2668-L2673)
  skips the listing entirely for agents that lack the Skill tool.
- `sentSkillNames` per-agent tracking
  ([lines 2699-2704](cc-code/restored-src/src/utils/attachments.ts#L2699-L2704)) — a skill is
  announced once, not re-sent every turn.

### Answer D: semantic retrieval as the escape valve

`EXPERIMENTAL_SKILL_SEARCH`
([attachments.ts:88-100](cc-code/restored-src/src/utils/attachments.ts#L88-L100)) replaces
enumeration with **retrieval**: a small side-model (Haiku or an embedding model — the comment
cites "AKI@250ms / Haiku@573ms") looks at the current context and returns only relevant skills
as a `skill_discovery` attachment.

When it's on, the static listing shrinks to bundled + MCP only
([attachments.ts:2692-2697](cc-code/restored-src/src/utils/attachments.ts#L2692-L2697)) —
"bundled + MCP are small and intent-signaled; user/project/plugin skills go through discovery."

Two operational details worth copying:

**Discovery is triggered by typed signals, not "every turn."** There's a `DiscoverySignal`
type; the two visible signals are `turn-0 user input` (blocking,
`getTurnZeroSkillDiscovery`) and `write-pivot detection` mid-turn
([query.ts:323-335](cc-code/restored-src/src/query.ts#L323-L335)). The comment records why:
the old always-on path "found nothing in prod **97%** of the time."

**Latency is hidden under existing work.** `startSkillDiscoveryPrefetch` fires *before* the
model call and is awaited *after* tools complete
([query.ts:1620-1628](cc-code/restored-src/src/query.ts#L1620-L1628)) — 250-570ms hidden
inside a 2-30s turn, "should be >98%." The only place it blocks is turn 0, "the one signal
where there's no prior work to hide under."

There's also a documented negative result: skill discovery deliberately does **not** fire on
plan exit
([attachments.ts:1268-1270](cc-code/restored-src/src/utils/attachments.ts#L1268-L1270)) — "By
the time the plan is written, it's too late; the model should have had relevant skills WHILE
planning."

### Recommended architecture for a growing, conflict-prone skill library

1. **Budget first.** Cap the always-visible listing (1% of context is the reference point) and
   cap per-entry descriptions (~250 chars). Force yourself to choose.
2. **Scope by `paths:`** — the highest-leverage fix for mutually-exclusive domain skills.
   Conflicting skills should be *mutually invisible*, not competing in one list.
3. **Defer the long tail.** Once MCP tool definitions exceed ~10% of context, switch to a
   search/reference model where the model sees names and pulls definitions on demand.
4. **Persist discovery across compaction.** Otherwise you pay the search round-trip repeatedly.
5. **Add semantic retrieval only when 1-3 aren't enough**, trigger it on *specific signals*,
   and **prefetch it under existing latency**.
6. **Announce as deltas**, never as a full re-listing, so the prompt cache survives.
7. **Instrument the match rate.** The `hidden_by_main_turn` flag and the 97%-no-op finding are
   what justified redesigning the trigger. If you can't measure how often discovery fires and
   how often it's used, you can't tune it.

---

## 4. Multi-agent / parallel / team strategy

There are **four distinct concurrency models**, and they're not variations of one thing — they
differ in isolation boundary, communication channel, and lifecycle.

### Level 1: intra-turn tool parallelism

[StreamingToolExecutor](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts). The
governing predicate is per-tool and **input-dependent**:

```ts
isConcurrencySafe(input: z.infer<Input>): boolean
```

Defaults **fail-closed to `false`**
([Tool.ts:750](cc-code/restored-src/src/Tool.ts#L750): "assume not safe"), alongside
`isReadOnly → false` and `isDestructive → false`. In a harness, an unmarked tool running in
parallel is a data race; unmarked-and-serial is just slow.

Scheduling: safe tools run in parallel with other safe tools; an unsafe tool runs exclusively;
the queue walk **`break`s** rather than skipping past a blocked unsafe tool, preserving order.
Results are buffered and emitted in receipt order.

Note `AgentTool.isConcurrencySafe()` returns **true**
([AgentTool.tsx:1273](cc-code/restored-src/src/tools/AgentTool/AgentTool.tsx#L1273)) — spawning
N subagents in one assistant message runs them all in parallel. The cheapest fan-out in the
system.

### Level 2: in-process subagents (fork)

`AgentTool` → [runAgent.ts](cc-code/restored-src/src/tools/AgentTool/runAgent.ts), an
`async function*` that runs a **nested query loop** in the same process.

- **Own context window.** The isolation is contextual, not process-level — the point is that
  the subagent's exploration doesn't pollute the parent's context. Only its final report returns.
- **Own tool set / model / permissions**, from the agent definition.
- **`agentId` is the routing key** for everything downstream: notification queue scoping,
  transcript sidechain, task summary gating.
- **Depth tracked** via `queryTracking: { chainId, depth }`
  ([query.ts:347-355](cc-code/restored-src/src/query.ts#L347-L355)) — every analytics event
  carries both, so you can reconstruct the spawn tree.
- Skill contents load **concurrently** at startup
  ([runAgent.ts:617](cc-code/restored-src/src/tools/AgentTool/runAgent.ts#L617)).

### Level 3: background agents (async fan-out)

`run_in_background: true`
([AgentTool.tsx:87](cc-code/restored-src/src/tools/AgentTool/AgentTool.tsx#L87)) detaches the
agent from the parent's turn.

**Deliberate abort-tree severing**
([AgentTool.tsx:694](cc-code/restored-src/src/tools/AgentTool/AgentTool.tsx#L694)):

> "Don't link to parent's abort controller — background agents should [survive]"

The one place the cancellation hierarchy is intentionally cut.

**Foreground→background promotion via race**
([AgentTool.tsx:808-890](cc-code/restored-src/src/tools/AgentTool/AgentTool.tsx#L808-L890)). The
agent registers as a foreground task *immediately*, then the loop races the next message
against a `backgroundSignal`:

```ts
backgroundPromise = registration.backgroundSignal.then(() => ({ type: 'background' }))
```

After `PROGRESS_THRESHOLD_MS = 2000` a hint appears offering to background it. The decision is
deferred to the point where it's informed — you know it's slow. `CLAUDE_AUTO_BACKGROUND_TASKS`
enables automatic promotion.

**Lifecycle constraints are enforced, with messages.** In-process teammates cannot spawn
background agents
([AgentTool.tsx:278-279, 361-362](cc-code/restored-src/src/tools/AgentTool/AgentTool.tsx#L278-L279))
— checked both for the explicit flag *and* for agent definitions with `background: true`. The
teammate's own lifecycle can't outlive its parent, so an orphaned background child would leak.

### Level 4: teams / swarm (multi-process peers)

The genuinely different model. [utils/swarm/](cc-code/restored-src/src/utils/swarm/) +
[tasks/InProcessTeammateTask/](cc-code/restored-src/src/tasks/InProcessTeammateTask/) +
`TeamCreateTool` / `SendMessageTool`.

**Backends are pluggable** ([swarm/backends/](cc-code/restored-src/src/utils/swarm/backends/)):
`InProcessBackend`, `TmuxBackend`, `ITermBackend` — teammates as panes in your terminal, or
in-process. Same protocol, different substrate.

**Peers, not a tree.** Unlike subagents, teammates are addressable by **name** and message each
other laterally. [SendMessageTool](cc-code/restored-src/src/tools/SendMessageTool/SendMessageTool.ts)
supports:

- direct send → "Message sent to {name}'s inbox"
- **broadcast** → `target: '@team'`
- **request/response** with a correlation id (`generateRequestId`)
- **shutdown request/approval** — a teammate can't be unilaterally killed; there's a negotiated
  protocol

**Communication is durable files with locks**, not in-memory channels
([teammateMailbox.ts](cc-code/restored-src/src/utils/teammateMailbox.ts)) — so it works across
processes, survives restarts, and is inspectable. Messages carry a `read` flag;
`readUnreadMessages` is the poll.

**Shared team state** beyond messages: `services/teamMemorySync`, `leaderPermissionBridge.ts`
and `permissionSync.ts` (permission decisions propagate — you don't want to approve the same
thing in five panes), `reconnection.ts` (teammates rejoin after disconnect).

### Comparison

| | Tool parallel | Subagent | Background agent | Teammate |
|---|---|---|---|---|
| Isolation | none (shared ctx) | own context | own context | own context + often own process |
| Topology | flat batch | tree (parent waits) | tree (detached) | **graph of peers** |
| Comms | tool results | final report only | task notification | mailbox: direct/broadcast/request |
| Lifecycle | within turn | within parent turn | outlives turn | outlives session |
| Cancellation | abort tree | abort tree | **severed** | shutdown protocol |
| Cost | ~0 | 1 context | 1 context | 1 context + process |

### Guidance

- **Default to Level 1.** Marking tools `isConcurrencySafe` is nearly free and gets most real
  speedup.
- **Level 2 when context is the bottleneck** — a search that reads 40 files should not put 40
  files in your main context.
- **Level 3 when latency is the bottleneck** and the result isn't needed to continue. Note the
  promotion pattern: start foreground, offer background at 2s.
- **Level 4 only for genuinely long-lived collaborators** with lateral communication. The cost
  is real: file locks, permission sync, reconnection, shutdown negotiation.
- **Whatever level: propagate a chain id and depth on every event.** `queryTracking` is what
  makes a multi-agent system debuggable. Without it you have concurrent logs and no causality.

---

## 5. Long-running task management

The task subsystem ([tasks/](cc-code/restored-src/src/tasks/),
[utils/task/](cc-code/restored-src/src/utils/task/)) generalizes over **seven** task types
sharing one lifecycle: `LocalShellTask`, `LocalAgentTask`, `RemoteAgentTask`,
`InProcessTeammateTask`, `LocalWorkflowTask`, `MonitorMcpTask`, `DreamTask`.

### Principle 1: output goes to disk, never to memory

[diskOutput.ts](cc-code/restored-src/src/utils/task/diskOutput.ts):

```ts
export const MAX_TASK_OUTPUT_BYTES = 5 * 1024 * 1024 * 1024   // 5GB disk cap
const DEFAULT_MAX_READ_BYTES = 8 * 1024 * 1024                // 8MB per read
```

A `DiskTaskOutput` class streams to a file. The model/UI reads **deltas by byte offset**:

```ts
export async function getTaskOutputDelta(taskId, offset, maxBytes = DEFAULT_MAX_READ_BYTES)
// "Reads only from the byte offset, up to maxBytes — never loads the full file."
```

Three properties fall out: memory is O(1) regardless of task duration; each poll delivers only
*new* output; and a 5GB cap with an in-band truncation marker means a runaway `yes` loop fills a
file instead of the heap.

`AppState` holds an `outputOffset` per task; `applyTaskOffsetsAndEvictions`
([framework.ts:213-217](cc-code/restored-src/src/utils/task/framework.ts#L213-L217)) merges
offset patches against **fresh** `prev.tasks`, not a pre-await snapshot — "so concurrent status
transitions aren't clobbered." Classic async state-merge bug.

### Principle 2: push, don't poll — with a slow poll as backstop

`POLL_INTERVAL_MS = 1000`
([framework.ts:22](cc-code/restored-src/src/utils/task/framework.ts#L22)) exists, but completion
is **pushed**. And there's an explicit warning against doing both
([framework.ts:199-203](cc-code/restored-src/src/utils/task/framework.ts#L199-L203)):

> "Completed tasks are NOT notified here — each task type handles its own completion
> notification via `enqueuePendingNotification()`. Generating attachments here would race with
> those per-type callbacks, causing **dual delivery** (one inline attachment + one separate API
> turn)."

Exactly one component owns each notification.

### Principle 3: a priority queue mediates task output vs. user input

[messageQueueManager.ts](cc-code/restored-src/src/utils/messageQueueManager.ts). Everything —
user input, task notifications, orphaned permissions — goes through one priority queue:

- `enqueue()` defaults to **`'next'`**
- `enqueuePendingNotification()` defaults to **`'later'`** — "so user input is processed first"
- FIFO within a priority level

The query loop drains it **mid-turn**
([query.ts:1566-1578](cc-code/restored-src/src/query.ts#L1566-L1578)), converting queued items
into attachments so the model can react without waiting for the turn to end. Two filters worth
noting:

```ts
const sleepRan = toolUseBlocks.some(b => b.name === SLEEP_TOOL_NAME)
getCommandsByMaxPriority(sleepRan ? 'later' : 'next')
```

If the model called Sleep, it's *explicitly waiting* — so drain `'later'` items too. Otherwise
only `'next'`.

And **agent scoping** ([query.ts:1560-1578](cc-code/restored-src/src/query.ts#L1560-L1578)): the
queue is a process-global singleton shared by coordinator and all in-process subagents. Main
thread drains `agentId === undefined`; a subagent drains only
`mode === 'task-notification' && cmd.agentId === currentAgentId`. Subagents **never** see the
user prompt stream, "even if someone stamps an agentId on one."

Slash commands are excluded from mid-turn drain — they must go through `processSlashCommand`
after the turn, not be sent to the model as text.

### Principle 4: notifications are a structured XML protocol

[LocalShellTask.tsx:79-89](cc-code/restored-src/src/tasks/LocalShellTask/LocalShellTask.tsx#L79-L89):

```xml
<task_notification>
  <task_id>…</task_id>
  <tool_use_id>…</tool_use_id>
  <output_file>…</output_file>
  <summary>…</summary>
</task_notification>
```

Note the deliberate **absence** of `<status>` for progress pings:

> "No `<status>` tag — print.ts treats `<status>` as a terminal signal and an unknown value
> falls through to 'completed', **falsely closing the task** for SDK consumers."

The presence/absence of a field is load-bearing. If you build this, define terminal vs.
non-terminal notifications explicitly rather than by convention.

### Principle 5: watchdogs for the failure mode that isn't failure

The best idea in this subsystem —
[`startStallWatchdog`](cc-code/restored-src/src/tasks/LocalShellTask/LocalShellTask.tsx#L46-L104).
A background command blocked on an interactive prompt never exits, produces no error, and hangs
forever. The watchdog:

1. `stat()`s the output file periodically; tracks last-growth time.
2. If output hasn't grown for `STALL_THRESHOLD_MS` (~45s), tails the last bytes.
3. Runs `looksLikePrompt()` — regex match on the **last line** against known prompt patterns.
4. If it looks like a prompt, fires a one-shot notification with **actionable remediation**:
   *"Kill this task and re-run with piped input (e.g. `echo y | command`) or a non-interactive
   flag."*
5. If not a prompt, **resets `lastGrowth`** so the next check is 45s out — "instead of
   re-reading the tail on every tick."

Two correctness details: `timer.unref()` so the watchdog never keeps the process alive, and the
`cancelled` flag is **latched before** any async-visible side effect so an overlapping tick bails.

There's a symmetric input-side check, `peekForStdinData` in
[utils/process.ts](cc-code/restored-src/src/utils/process.ts#L50).

### Principle 6: idempotent, atomic completion

[`enqueueShellNotification`](cc-code/restored-src/src/tasks/LocalShellTask/LocalShellTask.tsx#L105-L120):

> "Atomically check and set notified flag to prevent duplicate notifications. If the task was
> already marked as notified (e.g., by `TaskStopTool`), skip enqueueing."

The check-and-set happens **inside** the `updateTaskState` updater, so it's atomic w.r.t. the
state store. Multiple paths can complete a task (natural exit, user kill, tool-driven stop);
exactly one notification must be emitted.

Related: `registerTask` detects re-registration (resume) and carries forward UI state —
`retain`, `startTime`, `messages`, `diskLoaded` — then **skips the `task_started` SDK event** to
avoid double-emit
([framework.ts:88-116](cc-code/restored-src/src/utils/task/framework.ts#L88-L116)).

### Principle 7: bounded lifecycle with grace periods

```ts
export const STOPPED_DISPLAY_MS = 3_000   // killed tasks visible before eviction
export const PANEL_GRACE_MS = 30_000      // terminal agent tasks in coordinator panel
```

Plus `isBackgroundTask` ([tasks/types.ts:37-48](cc-code/restored-src/src/tasks/types.ts#L37-L48))
distinguishing *running* from *backgrounded*, `evictTaskOutput` / `cleanupTaskOutput` for disk
reclamation, and the whole thing registered with the cleanup registry so shutdown reclaims it.

### Principle 8: a tool surface for the model

`TaskCreateTool`, `TaskListTool`, `TaskGetTool`, `TaskOutputTool`, `TaskUpdateTool`,
`TaskStopTool`. The model manages its own long-running work — checks status, reads incremental
output, kills stuck tasks. Note `TaskOutputTool.isConcurrencySafe(_input) → true`
([TaskOutputTool.tsx:160](cc-code/restored-src/src/tools/TaskOutputTool/TaskOutputTool.tsx#L160)):
reading task output is a pure read, so status polls parallelize.

### Checklist

1. **Output to disk from byte zero**, read by offset. Never buffer unbounded output in memory.
2. **Hard cap on stored output** with an in-band truncation marker.
3. **Push completion, poll slowly as backstop** — and give each notification exactly one owner.
4. **Priority-queue task output behind user input**, drained mid-turn.
5. **Scope notifications by agent id** if you have subagents sharing a queue.
6. **Structured notification protocol** with an explicit terminal/non-terminal distinction.
7. **Stall watchdog** — the hang-with-no-error is the dominant real failure mode, and detecting
   it is cheap (`stat` + tail + regex).
8. **Idempotent completion** with an atomic notified-flag.
9. **`unref()` every background timer.**
10. **Grace periods before eviction**, so users and the model can still read what happened.

---

## The one idea underneath all five

Every subsystem here follows the same shape: **make the implicit explicit, then bound it.**

Errors become typed messages instead of exceptions. Loop continuation becomes a named
`transition` instead of recursion. Tool safety becomes a declared `isConcurrencySafe` instead of
an assumption. Skill relevance becomes a `paths:` predicate instead of a description the model
has to disambiguate. Task completion becomes a protocol field instead of "the process exited."

And every one of them gets a bound: 3 recovery attempts, 3 compaction failures, 1% of context for
skills, 10% before deferring tools, 5GB of output, 45s to stall, 5s to shut down.

If you're building a harness, those two moves — **name the state, then cap it** — are what
separate a loop that works in a demo from one that survives a 3,000-turn session.


