I've now traced the harness end to end. Here's a full explanation of `cc-code/restored-src/src`.

> Note: this tree is a **restored (de-minified) source dump of Claude Code itself** — the CLI you're talking to right now. ~1,900 TS/TSX files. Everything below is grounded in the files I read; I link the important ones.

---

## 1. Pipeline / workflow

There are two entry paths that both converge on a single **agentic loop** (`query()`).

```
                        ┌─────────────────────────────────────────────┐
  process start         │  entrypoints/cli.tsx  →  main()             │
  (`claude …`)  ───────▶│  fast-paths: --version, mcp, daemon, bridge,│
                        │  bg, ps … else → full CLI (main.tsx→setup)  │
                        └───────────────┬─────────────────────────────┘
                                        │
                 ┌──────────────────────┴───────────────────────┐
        interactive TTY                                    headless / -p / SDK
                 │                                                │
   replLauncher.launchRepl()                              cli/print.ts → ask()
   renders <App><REPL/></App> (Ink)                       → QueryEngine.submitMessage()
                 │                                                │
        REPL.onQuery → onQueryImpl                                │
                 └───────────────────────┬──────────────────────┘
                                         ▼
        ┌──────────────────────────────────────────────────────────────┐
        │  processUserInput()  — turn slash/bash/text into Message[]     │
        │   (utils/processUserInput/*)                                   │
        └───────────────────────────────┬──────────────────────────────┘
                                         ▼
        ╔══════════════  query() loop  (query.ts) — one iteration = one API turn ══════════╗
        ║ 1. Shape context: applyToolResultBudget → snip → microcompact →                  ║
        ║    contextCollapse → autocompact; token-budget / blocking-limit checks           ║
        ║ 2. deps.callModel(...)  → stream from Anthropic API (services/api/claude.ts)     ║
        ║ 3. yield assistant messages; collect tool_use blocks                             ║
        ║ 4. tool_use blocks?                                                              ║
        ║      ├─ YES → runTools()/StreamingToolExecutor:                                  ║
        ║      │        partition (concurrency) → canUseTool (permission) → tool.call()    ║
        ║      │        → tool_result → attachments → **loop back to step 1** (next turn)  ║
        ║      └─ NO  → Stop hooks → (continue | terminate)                                ║
        ╚══════════════════════════════════════════════╤═══════════════════════════════════╝
                                         │ yields Message / StreamEvent
                 ┌───────────────────────┴────────────────────────┐
        REPL: onQueryEvent → setState → Ink re-render      Headless: normalize → SDKMessage
                 │                                                │
        transcript persisted to <sessionId>.jsonl (sessionStorage.ts) in both paths
```

The loop is a single `while (true)` in [query.ts:307](cc-code/restored-src/src/query.ts#L307). Each iteration is one request/response to the model. It **recurses via `continue`** whenever there are tool results to feed back, and **returns a `Terminal`** when the model stops without calling a tool (or on error/abort/limit). This is the classic "LLM ⇄ tools" agent loop; almost all the code around it is context management, recovery, and permissions.

---

## 2. Tech stack & state

**Language / build:** TypeScript (ESM), bundled with **Bun**. The `feature('X')` calls (from `bun:bundle`) are compile-time flags used for dead-code elimination — that's why you see `feature('HISTORY_SNIP')`, `feature('COORDINATOR_MODE')` gating whole subsystems.

**UI: React — but rendered to the terminal, not a browser.** The `ink/` directory is a **vendored/forked copy of Ink**: a full `react-reconciler` host config ([ink/reconciler.ts](cc-code/restored-src/src/ink/reconciler.ts)) plus a Yoga flexbox layout engine ([ink/layout/](cc-code/restored-src/src/ink/layout/)) that paints `<Box>`/`<Text>` as ANSI to stdout. So React components (`screens/`, `components/`) render a TUI.

**State management: NOT Redux.** It's a hand-rolled minimal store:

- [state/store.ts](cc-code/restored-src/src/state/store.ts) — `createStore` is ~30 lines: `getState`, `setState(updater)`, `subscribe(listener)` over a `Set` of listeners (plus an `onChange` hook). That's the entire "store engine."
- [state/AppState.tsx](cc-code/restored-src/src/state/AppState.tsx) — wraps it in a React Context (`AppStateProvider`) and exposes `useAppState(selector)` built on **`useSyncExternalStore`** with `Object.is`-compared selector slices (so a component only re-renders when its slice changes). `useSetAppState()` returns the stable setter without subscribing.

So the equivalent of "Redux" here = **React Context + external store + selector subscriptions**, no reducers/actions/middleware, no `react-redux`.

**The "app state"** is one big `AppState` object defined in [state/AppStateStore.ts](cc-code/restored-src/src/state/AppStateStore.ts) (`getDefaultAppState()` at [line 456](cc-code/restored-src/src/state/AppStateStore.ts#L456)). It holds: `settings`, `mainLoopModel`, `toolPermissionContext`, `mcp` (clients/tools/commands), `plugins`, `tasks` (subagents), `todos`, `agentDefinitions`, `fileHistory`, `notifications`, `elicitation`, `speculation`, bridge/remote flags, `fastMode`, etc. It's a `DeepImmutable<…>` — updated by returning new objects from `setState`.

**What is persisted?** This is the key subtlety:

- **The store itself is in-memory and ephemeral.** `AppState` is created fresh each session via `getDefaultAppState()`; nothing of the reactive store is serialized to disk. So: *nothing "in Redux" is persisted.*
- **The conversation transcript is what's persisted** — separately from the store — as JSONL. [sessionStorage.ts](cc-code/restored-src/src/utils/sessionStorage.ts) writes messages to `~/.claude/projects/<project>/<sessionId>.jsonl` (`getTranscriptPath` at [line 202](cc-code/restored-src/src/utils/sessionStorage.ts#L202)); subagents get their own `agent-<id>.jsonl`. `recordTranscript()` / `flushSessionStorage()` are called throughout `QueryEngine` and this is what powers `--resume`.
- **Settings & config** persist as files (`settings.json` at user/project/local scopes, global config). They're *loaded into* `AppState.settings` at startup and kept in sync via `applySettingsChange` when the files change — but the source of truth on disk is the file, not the store.

Summary: **React ✔, Redux ✘.** State lives in a custom external store that is memory-only; durability comes from the JSONL transcript + settings/config files, which are mirrored into the store rather than persisted from it.

---

## 3. Structure of the harness

Layered, roughly outer→inner:

| Layer | Where | Role |
|---|---|---|
| **Entry / bootstrap** | [entrypoints/](cc-code/restored-src/src/entrypoints/), [main.tsx](cc-code/restored-src/src/main.tsx), [setup.ts](cc-code/restored-src/src/setup.ts), [cli/](cc-code/restored-src/src/cli/) | Arg parsing, fast-paths, config/auth, choose interactive vs headless |
| **Agent loop** | [query.ts](cc-code/restored-src/src/query.ts), [QueryEngine.ts](cc-code/restored-src/src/QueryEngine.ts), [query/](cc-code/restored-src/src/query/), [services/tools/](cc-code/restored-src/src/services/tools/) | The LLM⇄tools loop + turn/budget/error handling |
| **Tool abstraction** | [Tool.ts](cc-code/restored-src/src/Tool.ts), [tools.ts](cc-code/restored-src/src/tools.ts), [tools/](cc-code/restored-src/src/tools/) | `Tool` interface, registry, ~50 built-in tools (one dir each) |
| **State / contexts** | [state/](cc-code/restored-src/src/state/), [context/](cc-code/restored-src/src/context/) | The store + React contexts (mailbox, notifications, modals…) |
| **UI (TUI)** | [screens/REPL.tsx](cc-code/restored-src/src/screens/REPL.tsx), [components/](cc-code/restored-src/src/components/), [ink/](cc-code/restored-src/src/ink/) | React-to-terminal rendering |
| **Model / API** | [services/api/](cc-code/restored-src/src/services/api/) | Anthropic streaming, retries, usage/cost |
| **Context mgmt** | [services/compact/](cc-code/restored-src/src/services/compact/), [context.ts](cc-code/restored-src/src/context.ts), [utils/attachments.ts](cc-code/restored-src/src/utils/attachments.ts), [memdir/](cc-code/restored-src/src/memdir/) | Compaction, system/user context, CLAUDE.md, memory |
| **Permissions** | [hooks/useCanUseTool.tsx](cc-code/restored-src/src/hooks/useCanUseTool.tsx), [utils/permissions/](cc-code/restored-src/src/utils/permissions/) | Allow/deny/ask decisions, permission modes |
| **Lifecycle hooks** | [utils/hooks/](cc-code/restored-src/src/utils/hooks/) | User-configured PreToolUse/Stop/SessionStart… shell/prompt hooks |
| **Slash commands** | [commands/](cc-code/restored-src/src/commands/) (~85), [commands.ts](cc-code/restored-src/src/commands.ts) | `/model`, `/compact`, `/resume`, … |
| **MCP** | [services/mcp/](cc-code/restored-src/src/services/mcp/), [tools/MCPTool/](cc-code/restored-src/src/tools/MCPTool/) | External tool servers |
| **Subagents / tasks** | [tools/AgentTool/](cc-code/restored-src/src/tools/AgentTool/), [tasks/](cc-code/restored-src/src/tasks/), [coordinator/](cc-code/restored-src/src/coordinator/) | Spawning nested agents & background work |
| **Skills / plugins** | [skills/](cc-code/restored-src/src/skills/), [plugins/](cc-code/restored-src/src/plugins/) | Bundled skills, plugin loading |
| **Remote** | [bridge/](cc-code/restored-src/src/bridge/), [remote/](cc-code/restored-src/src/remote/), [cli/transports/](cc-code/restored-src/src/cli/transports/) | claude.ai bridge / remote sessions |
| **SDK** | [entrypoints/sdk/](cc-code/restored-src/src/entrypoints/sdk/), [QueryEngine.ts](cc-code/restored-src/src/QueryEngine.ts) | Programmatic (Agent SDK) surface |

The important design seam: **`query()` is UI-agnostic.** It takes a `ToolUseContext` ([Tool.ts:158](cc-code/restored-src/src/Tool.ts#L158)) full of injected callbacks (`getAppState`, `setAppState`, `setToolJSX?`, `addNotification?`…). The REPL wires those to real React state; the headless `QueryEngine` wires them to no-ops or SDK emitters. Same loop, two shells.

---

## 4. Main functions of the harness

**Agent loop core**
- `query()` / `queryLoop()` — [query.ts:219](cc-code/restored-src/src/query.ts#L219) — the master async generator; one iteration per model turn, `continue` to recurse on tool results, returns a `Terminal` reason (`completed`, `aborted_tools`, `max_turns`, `prompt_too_long`, `blocking_limit`…).
- `QueryEngine.submitMessage()` — [QueryEngine.ts:209](cc-code/restored-src/src/QueryEngine.ts#L209) — owns a conversation's lifecycle for the headless/SDK path: builds system prompt, drives `query()`, translates internal `Message`s → `SDKMessage`s, tracks usage/cost/denials, persists transcript. `ask()` ([line 1186](cc-code/restored-src/src/QueryEngine.ts#L1186)) is the one-shot wrapper.
- `onQueryImpl()` — [REPL.tsx:2661](cc-code/restored-src/src/screens/REPL.tsx#L2661) — the interactive counterpart; assembles system/user context, then `for await (event of query(...)) onQueryEvent(event)` at [line 2793](cc-code/restored-src/src/screens/REPL.tsx#L2793).

**Model call**
- `deps.callModel(...)` — invoked at [query.ts:659](cc-code/restored-src/src/query.ts#L659); implementation in [services/api/claude.ts](cc-code/restored-src/src/services/api/) — streams from the Anthropic API, handles fallback model, task budget, cache control, usage accumulation.

**Tool execution**
- `runTools()` — [services/tools/toolOrchestration.ts:19](cc-code/restored-src/src/services/tools/toolOrchestration.ts#L19) — partitions tool calls into **concurrent read-only batches vs serial write batches** (`partitionToolCalls`), runs up to `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` at once.
- `StreamingToolExecutor` — [services/tools/StreamingToolExecutor.ts](cc-code/restored-src/src/services/tools/StreamingToolExecutor.ts) — the newer path that starts running tools *while the model is still streaming* the next block.
- `Tool` interface & `buildTool()` — [Tool.ts:362](cc-code/restored-src/src/Tool.ts#L362) / [line 783](cc-code/restored-src/src/Tool.ts#L783) — every tool implements `call`, `inputSchema` (Zod), `checkPermissions`, `isConcurrencySafe`, `isReadOnly`, render methods, etc.
- Tool registry — `getAllBaseTools()` / `getTools()` / `assembleToolPool()` — [tools.ts:193](cc-code/restored-src/src/tools.ts#L193), [line 271](cc-code/restored-src/src/tools.ts#L271), [line 345](cc-code/restored-src/src/tools.ts#L345) — builds the tool list (feature-gated), filters by deny rules, merges MCP tools with cache-stable ordering.

**Permissions**
- `canUseTool` / `useCanUseTool` — [hooks/useCanUseTool.tsx:28](cc-code/restored-src/src/hooks/useCanUseTool.tsx#L28) — returns `{behavior: 'allow'|'deny'|'ask', …}`; in the REPL "ask" pushes onto a confirmation queue (UI dialog); in headless it's wrapped to record denials ([QueryEngine.ts:244](cc-code/restored-src/src/QueryEngine.ts#L244)).

**Input processing**
- `processUserInput()` — [utils/processUserInput/processUserInput.ts](cc-code/restored-src/src/utils/processUserInput/) — routes raw input to `processSlashCommand` / `processBashCommand` / `processTextPrompt`, producing `Message[]` and a `shouldQuery` flag.

**Context / prompt assembly**
- `fetchSystemPromptParts()` — [utils/queryContext.ts:44](cc-code/restored-src/src/utils/queryContext.ts#L44) — pulls the default system prompt + user context + system context.
- `getUserContext()` / `getSystemContext()` — [context.ts:155](cc-code/restored-src/src/context.ts#L155) / [line 116](cc-code/restored-src/src/context.ts#L116) — inject CLAUDE.md memory + today's date (user) and git status (system), memoized per conversation.

**Context window management** (the bulk of the pre-model code in `query.ts`)
- `autocompact` / `microcompact` / snip / contextCollapse — [services/compact/](cc-code/restored-src/src/services/compact/) — summarize or drop old history when nearing the token limit; `applyToolResultBudget()` ([utils/toolResultStorage.ts](cc-code/restored-src/src/utils/)) offloads oversized tool results to disk.

**Lifecycle hooks**
- `handleStopHooks()` — [query/stopHooks.ts](cc-code/restored-src/src/query/stopHooks.ts) — runs when the model stops; can block and force continuation.
- `executePostSamplingHooks()` / `executeStopFailureHooks()` — [utils/hooks/](cc-code/restored-src/src/utils/hooks/) — user-configured PreToolUse/PostToolUse/SessionStart/etc.

**Subagents**
- `AgentTool` → `runAgent()` — [tools/AgentTool/AgentTool.tsx:196](cc-code/restored-src/src/tools/AgentTool/AgentTool.tsx#L196), `runAgent` at [line 55](cc-code/restored-src/src/tools/AgentTool/AgentTool.tsx#L55) — a tool that spawns a **nested `query()`** with its own filtered tools/context (optionally backgrounded), so the whole harness is recursive.

**State & persistence**
- `createStore` / `useAppState` / `useSetAppState` — [state/store.ts](cc-code/restored-src/src/state/store.ts), [state/AppState.tsx:142](cc-code/restored-src/src/state/AppState.tsx#L142).
- `recordTranscript` / `flushSessionStorage` — [utils/sessionStorage.ts](cc-code/restored-src/src/utils/sessionStorage.ts) — the JSONL durability layer.

---

Want me to go deeper on any one area — e.g. a step-through of a single tool call (permission → execute → result → next turn), the compaction pipeline, or how subagents/tasks are scheduled?

---

I've now read into every layer. Here's the detailed breakdown of each row from the structure table — **theory** (the general idea), **implementation** (how this codebase does it), and **reference** (files/functions).

---

## 1. Entry / bootstrap

**Theory.** A CLI needs a tiny, cheap entry that parses argv, handles trivial/fast commands (`--version`) without loading the world, decides *which mode* to run (interactive TUI vs one-shot headless vs a background server), loads config/auth, then hands off. Keeping this thin matters for startup latency.

**Implementation.** [entrypoints/cli.tsx](cc-code/restored-src/src/entrypoints/cli.tsx) is the true process entry. Its `main()` ([line 33](cc-code/restored-src/src/entrypoints/cli.tsx#L33)) uses **all-dynamic imports** so each fast-path loads only what it needs: `--version` returns with zero further imports; then `--dump-system-prompt`, `--claude-in-chrome-mcp`, `daemon`, `remote-control`/`bridge`, `ps/logs/attach/kill` each short-circuit. Anything else falls through to the full CLI in the 800 KB [main.tsx](cc-code/restored-src/src/main.tsx), which calls `setup()` ([setup.ts:56](cc-code/restored-src/src/setup.ts#L56)) to initialize config, auth, analytics sinks, MCP, etc., then either `launchRepl()` (interactive) or the print path. `feature('X')` guards (from `bun:bundle`) let the bundler *delete* whole branches (daemon, bridge) from external builds.

**Reference.** [entrypoints/cli.tsx](cc-code/restored-src/src/entrypoints/cli.tsx), [setup.ts](cc-code/restored-src/src/setup.ts), [main.tsx](cc-code/restored-src/src/main.tsx), [replLauncher.tsx](cc-code/restored-src/src/replLauncher.tsx), [cli/print.ts](cc-code/restored-src/src/cli/print.ts).

---

## 2. Agent loop

**Theory.** The heart of any tool-using agent is a loop: send conversation to the model → model responds with text and/or tool calls → if tool calls, execute them and append results → repeat until the model answers with no tool call. Everything else (budgets, compaction, recovery) hangs off this skeleton.

**Implementation.** [query.ts](cc-code/restored-src/src/query.ts) is a single `while(true)` async generator ([queryLoop, line 241](cc-code/restored-src/src/query.ts#L241)). Each iteration: shape context → `deps.callModel(...)` (stream) → collect `tool_use` blocks → if present, run tools and `continue` (next turn); if absent, run Stop hooks and `return` a `Terminal`. Cross-iteration mutable data lives in one `State` object so the seven `continue` sites each just write `state = {...}`. I/O is injected via `QueryDeps` ([query/deps.ts](cc-code/restored-src/src/query/deps.ts)) so tests can swap `callModel`/compaction. Immutable per-turn snapshots are separated into `QueryConfig` ([query/config.ts](cc-code/restored-src/src/query/config.ts)). [QueryEngine.ts](cc-code/restored-src/src/QueryEngine.ts) wraps `query()` for the headless/SDK path (one engine per conversation; `submitMessage()` per turn, translating internal `Message`s → `SDKMessage`s).

**Reference.** `queryLoop` [query.ts:241](cc-code/restored-src/src/query.ts#L241); `QueryEngine.submitMessage` [QueryEngine.ts:209](cc-code/restored-src/src/QueryEngine.ts#L209); [query/deps.ts](cc-code/restored-src/src/query/deps.ts), [query/config.ts](cc-code/restored-src/src/query/config.ts), [query/tokenBudget.ts](cc-code/restored-src/src/query/tokenBudget.ts), [query/stopHooks.ts](cc-code/restored-src/src/query/stopHooks.ts).

---

## 3. Tool abstraction

**Theory.** Tools are the agent's hands. A uniform interface lets the loop treat every capability the same: a name, an input schema (for the model + validation), a permission check, an executor, and rendering. Schema-first design means the same object produces the API tool definition, runtime validation, and UI.

**Implementation.** [Tool.ts](cc-code/restored-src/src/Tool.ts) defines the `Tool` interface ([line 362](cc-code/restored-src/src/Tool.ts#L362)): `inputSchema` (Zod), `call()`, `checkPermissions()`, `validateInput()`, plus behavioral predicates (`isReadOnly`, `isConcurrencySafe`, `isDestructive`) that the loop uses to decide parallelism and permission flow, and many `renderX()` methods for the TUI. `buildTool()` ([line 783](cc-code/restored-src/src/Tool.ts#L783)) fills fail-closed defaults (`isReadOnly→false`, `isConcurrencySafe→false`). Each tool is a directory under [tools/](cc-code/restored-src/src/tools/) (BashTool, FileEditTool, AgentTool, …). [tools.ts](cc-code/restored-src/src/tools.ts) is the registry: `getAllBaseTools()` ([line 193](cc-code/restored-src/src/tools.ts#L193)) lists them (feature-gated), `getTools()` filters by permission deny-rules and mode, and `assembleToolPool()` ([line 345](cc-code/restored-src/src/tools.ts#L345)) merges MCP tools in a **cache-stable sort order** (built-ins as a contiguous prefix so the server's prompt cache isn't invalidated).

**Reference.** [Tool.ts](cc-code/restored-src/src/Tool.ts) (`Tool`, `ToolUseContext` at [line 158](cc-code/restored-src/src/Tool.ts#L158), `buildTool`), [tools.ts](cc-code/restored-src/src/tools.ts), [tools/](cc-code/restored-src/src/tools/).

---

## 4. State / contexts

**Theory.** A TUI app needs shared reactive state (current model, permission mode, running tasks…) that many components read and few mutate, without prop-drilling and without re-rendering everything on every change. The standard modern pattern is an external store + selector subscriptions.

**Implementation.** Not Redux — a ~30-line store: [state/store.ts](cc-code/restored-src/src/state/store.ts) `createStore` gives `getState / setState(updater) / subscribe`, notifying a `Set` of listeners and skipping no-op updates (`Object.is`). [state/AppState.tsx](cc-code/restored-src/src/state/AppState.tsx) provides it via React Context and exposes `useAppState(selector)` on **`useSyncExternalStore`** ([line 142](cc-code/restored-src/src/state/AppState.tsx#L142)) so a component re-renders only when its selected slice changes; `useSetAppState()` returns the setter without subscribing. The single `AppState` shape lives in [state/AppStateStore.ts](cc-code/restored-src/src/state/AppStateStore.ts) (`getDefaultAppState` [line 456](cc-code/restored-src/src/state/AppStateStore.ts#L456)). Alongside the store, cross-cutting UI concerns use ordinary React contexts in [context/](cc-code/restored-src/src/context/) (notifications, modal, mailbox, overlay, stats) — these are ephemeral view concerns, kept out of the main store.

**Reference.** [state/store.ts](cc-code/restored-src/src/state/store.ts), [state/AppState.tsx](cc-code/restored-src/src/state/AppState.tsx), [state/AppStateStore.ts](cc-code/restored-src/src/state/AppStateStore.ts), [context/notifications.tsx](cc-code/restored-src/src/context/notifications.tsx).

---

## 5. UI (TUI)

**Theory.** Rendering React to a terminal means replacing the DOM host with a custom renderer: React reconciles a tree of `<Box>`/`<Text>`, a flexbox engine computes coordinates, and the result is diffed and written as ANSI escape codes to stdout. Same declarative model as web React, different host.

**Implementation.** [ink/](cc-code/restored-src/src/ink/) is a **vendored/forked Ink**: [ink/reconciler.ts](cc-code/restored-src/src/ink/reconciler.ts) builds a `react-reconciler` host config ([line 224](cc-code/restored-src/src/ink/reconciler.ts#L224)) whose `resetAfterCommit` triggers layout + paint; [ink/layout/](cc-code/restored-src/src/ink/layout/) wraps Yoga for flexbox; [ink/termio/](cc-code/restored-src/src/ink/termio/) parses/emits ANSI (SGR/CSI/OSC). On top sit the actual screens: [screens/REPL.tsx](cc-code/restored-src/src/screens/REPL.tsx) (~895 KB — the whole interactive experience; `onQueryImpl` at [line 2661](cc-code/restored-src/src/screens/REPL.tsx#L2661) is where it drives `query()`) and the widget library in [components/](cc-code/restored-src/src/components/) (PromptInput, message list, dialogs, permission prompts).

**Reference.** [ink/reconciler.ts](cc-code/restored-src/src/ink/reconciler.ts), [ink/ink.tsx](cc-code/restored-src/src/ink/ink.tsx), [ink/layout/](cc-code/restored-src/src/ink/layout/), [screens/REPL.tsx](cc-code/restored-src/src/screens/REPL.tsx), [components/](cc-code/restored-src/src/components/).

---

## 6. Model / API

**Theory.** The harness must turn its internal message list into a provider request, stream the response back as events (text deltas, tool calls, usage), and be resilient: retries, fallback models, prompt-cache breakpoints to cut cost, and token accounting.

**Implementation.** [services/api/claude.ts](cc-code/restored-src/src/services/api/claude.ts) is the boundary. `queryModelWithStreaming()` ([line 752](cc-code/restored-src/src/services/api/claude.ts#L752)) is what `query()` calls as `deps.callModel`; it wraps the internal `queryModel()` generator in a "VCR" layer (record/replay fixtures for tests). Around it: `addCacheBreakpoints()` ([line 3063](cc-code/restored-src/src/services/api/claude.ts#L3063)) and `getCacheControl()` insert `cache_control` markers for prompt caching; `withRetry` ([services/api/withRetry.ts](cc-code/restored-src/src/services/api/withRetry.ts)) handles retries and raises `FallbackTriggeredError` (which `query.ts` catches to switch to `fallbackModel`); `updateUsage`/`accumulateUsage` track tokens; `configureTaskBudgetParams` handles the server-side task budget. Message translation (`userMessageToMessageParam`, `assistantMessageToMessageParam`) converts internal `Message`s to the Anthropic SDK shape.

**Reference.** `queryModelWithStreaming` [claude.ts:752](cc-code/restored-src/src/services/api/claude.ts#L752), `addCacheBreakpoints` [line 3063](cc-code/restored-src/src/services/api/claude.ts#L3063), [services/api/withRetry.ts](cc-code/restored-src/src/services/api/withRetry.ts), [services/api/errors.ts](cc-code/restored-src/src/services/api/errors.ts), [services/api/logging.ts](cc-code/restored-src/src/services/api/logging.ts).

---

## 7. Context management

**Theory.** Context windows are finite. A long agent session must (a) inject the right *ambient* context (system prompt, project memory, git state, file changes), and (b) shrink history before it overflows — by summarizing ("compact"), dropping stale tool results ("microcompact"), or collapsing — while preserving correctness (e.g. thinking-block rules, tool_use/tool_result pairing).

**Implementation.** Ambient context: [context.ts](cc-code/restored-src/src/context.ts) builds `getUserContext()` (CLAUDE.md memory + date, [line 155](cc-code/restored-src/src/context.ts#L155)) and `getSystemContext()` (git status, [line 116](cc-code/restored-src/src/context.ts#L116)), both memoized per conversation; [utils/attachments.ts](cc-code/restored-src/src/utils/attachments.ts) injects file-change/memory/skill attachments each turn; [memdir/](cc-code/restored-src/src/memdir/) handles the memory directory. Shrinking: [services/compact/](cc-code/restored-src/src/services/compact/) — `autoCompactIfNeeded` fires near the threshold (`getAutoCompactThreshold`, `AUTOCOMPACT_BUFFER_TOKENS = 13k`, [autoCompact.ts](cc-code/restored-src/src/services/compact/autoCompact.ts)), `microcompactMessages` drops old tool outputs by id, plus optional snip/contextCollapse/reactiveCompact recovery paths. These are the calls at the top of every `query()` iteration ([query.ts:379–467](cc-code/restored-src/src/query.ts#L379)); oversized tool results are offloaded to disk via `applyToolResultBudget`.

**Reference.** [context.ts](cc-code/restored-src/src/context.ts), [services/compact/autoCompact.ts](cc-code/restored-src/src/services/compact/autoCompact.ts), [services/compact/microCompact.ts](cc-code/restored-src/src/services/compact/microCompact.ts), [services/compact/compact.ts](cc-code/restored-src/src/services/compact/compact.ts), [utils/attachments.ts](cc-code/restored-src/src/utils/attachments.ts), [memdir/](cc-code/restored-src/src/memdir/).

---

## 8. Permissions

**Theory.** An autonomous tool-runner is dangerous, so every tool call passes a gate that returns **allow / deny / ask**. Decisions come from a rule system (allow/deny/ask rules, by tool and argument pattern), a permission *mode* (default, plan, acceptEdits, bypass, auto), and per-tool logic. In non-interactive modes an AI classifier can stand in for the human.

**Implementation.** The loop calls `canUseTool` before executing any tool. In the REPL that's [hooks/useCanUseTool.tsx](cc-code/restored-src/src/hooks/useCanUseTool.tsx) — an "ask" pushes a confirmation onto a UI queue and awaits the dialog. The rule engine is [utils/permissions/permissions.ts](cc-code/restored-src/src/utils/permissions/permissions.ts): `hasPermissionsToUseTool` ([line 473](cc-code/restored-src/src/utils/permissions/permissions.ts#L473)) resolves deny → ask → allow rules (`getDenyRules`/`getAskRules`/`getAllowRules`), applies mode transforms (`dontAsk` turns ask→deny; `auto` routes ask through the [yoloClassifier](cc-code/restored-src/src/utils/permissions/yoloClassifier.ts)), and tracks consecutive denials to fall back to prompting. Modes live in [utils/permissions/PermissionMode.ts](cc-code/restored-src/src/utils/permissions/PermissionMode.ts); rule parsing in [permissionRuleParser.ts](cc-code/restored-src/src/utils/permissions/permissionRuleParser.ts). The active rules/mode sit in `AppState.toolPermissionContext`.

**Reference.** [hooks/useCanUseTool.tsx](cc-code/restored-src/src/hooks/useCanUseTool.tsx), `hasPermissionsToUseTool` [permissions.ts:473](cc-code/restored-src/src/utils/permissions/permissions.ts#L473), [utils/permissions/](cc-code/restored-src/src/utils/permissions/) (PermissionMode, PermissionRule, yoloClassifier, denialTracking).

---

## 9. Lifecycle hooks

**Theory.** Users want to run their own logic at defined moments — validate/block a tool before it runs, react after, gate a Stop, seed a session. A hook system defines named events and runs user-configured shell/HTTP/prompt commands at each, letting the result influence control flow (e.g. block).

**Implementation.** The event set is `HOOK_EVENTS` in [entrypoints/sdk/coreTypes.ts:25](cc-code/restored-src/src/entrypoints/sdk/coreTypes.ts#L25): `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `Notification`, `UserPromptSubmit`, `SessionStart`, `SessionEnd`, `Stop`, `StopFailure`, `SubagentStop`, `PreCompact`, etc. [utils/hooks.ts](cc-code/restored-src/src/utils/hooks.ts) (huge) implements the executors — `executePreToolHooks` ([line 3394](cc-code/restored-src/src/utils/hooks.ts#L3394)), `executePostToolHooks`, `executeStopHooks`, `executeUserPromptSubmitHooks`, `executeSessionStartHooks` — plus `getMatchingHooks` to match configured hooks against a tool/pattern. `query()` wires them in: Stop hooks via `handleStopHooks` ([query/stopHooks.ts](cc-code/restored-src/src/query/stopHooks.ts)) can force continuation, and PreToolUse runs inside tool execution. Execution mechanics (shell/prompt/http, async registry) are in [utils/hooks/](cc-code/restored-src/src/utils/hooks/).

**Reference.** [utils/hooks.ts](cc-code/restored-src/src/utils/hooks.ts), [utils/hooks/hookEvents.ts](cc-code/restored-src/src/utils/hooks/hookEvents.ts), [query/stopHooks.ts](cc-code/restored-src/src/query/stopHooks.ts), `HOOK_EVENTS` [coreTypes.ts:25](cc-code/restored-src/src/entrypoints/sdk/coreTypes.ts#L25).

---

## 10. Slash commands

**Theory.** Interactive users need meta-actions that aren't messages to the model (`/model`, `/compact`, `/resume`, `/help`). A command registry maps names to handlers; some produce local output, some mutate state, some are "prompt" commands that expand into a model turn.

**Implementation.** The `Command` type and registry are in [commands.ts](cc-code/restored-src/src/commands.ts): `builtInCommandNames`, `findCommand`/`getCommand` ([line 688](cc-code/restored-src/src/commands.ts#L688)), availability filtering (`meetsAvailabilityRequirement`, `REMOTE_SAFE_COMMANDS`, `BRIDGE_SAFE_COMMANDS`). Each command is a directory under [commands/](cc-code/restored-src/src/commands/) (~85: agents, compact, config, resume, review, model, mcp, plugin…). At input time, [utils/processUserInput/processSlashCommand.tsx](cc-code/restored-src/src/utils/processUserInput/processSlashCommand.tsx) dispatches; commands can be local (return `shouldQuery=false` with output messages) or expand into prompts. Skills and MCP prompts are surfaced as commands too (`getSkillToolCommands`, `getMcpSkillCommands`).

**Reference.** [commands.ts](cc-code/restored-src/src/commands.ts), [commands/](cc-code/restored-src/src/commands/), [utils/processUserInput/processSlashCommand.tsx](cc-code/restored-src/src/utils/processUserInput/processSlashCommand.tsx).

---

## 11. MCP (Model Context Protocol)

**Theory.** MCP is the open standard for plugging in *external* tool servers (local subprocess over stdio, or remote over HTTP/SSE/WebSocket). The harness discovers a server's tools/resources/prompts, normalizes them into its own tool interface, and manages connection lifecycle and auth.

**Implementation.** [services/mcp/](cc-code/restored-src/src/services/mcp/): `MCPConnectionManager.tsx` and `useManageMCPConnections.ts` own connecting to configured servers and populating `AppState.mcp` (`clients`, `tools`, `commands`, `resources`); `client.ts` speaks the protocol; `config.ts` reads server definitions; `auth.ts`/`oauthPort.ts` handle OAuth; `normalization.ts` maps MCP tools to the harness `Tool` shape (prefixed `mcp__server__tool`); `InProcessTransport.ts`/`SdkControlTransport.ts` are transport variants. The generic wrapper tools that call into these are [tools/MCPTool/](cc-code/restored-src/src/tools/MCPTool/), plus `ListMcpResourcesTool`/`ReadMcpResourceTool`. Assembled into the pool via `assembleToolPool()` (row 3).

**Reference.** [services/mcp/](cc-code/restored-src/src/services/mcp/) (`MCPConnectionManager.tsx`, `useManageMCPConnections.ts`, `client.ts`, `normalization.ts`, `types.ts`), [tools/MCPTool/](cc-code/restored-src/src/tools/MCPTool/).

---

## 12. Subagents / tasks

**Theory.** For big jobs, a leader agent spawns *subagents* with their own context window and tool subset (isolation + parallelism), or backgrounds long-running work (shells, remote agents) as tracked "tasks." Since the loop is a generator, a subagent is just a nested invocation of the same loop.

**Implementation.** The `AgentTool` ([tools/AgentTool/AgentTool.tsx](cc-code/restored-src/src/tools/AgentTool/AgentTool.tsx)) calls `runAgent()` ([tools/AgentTool/runAgent.ts:248](cc-code/restored-src/src/tools/AgentTool/runAgent.ts#L248)), which builds an isolated context via `createSubagentContext` (its `setAppState` is a no-op so nested agents can't clobber root state) and drives a fresh `query({...})` ([line 748](cc-code/restored-src/src/tools/AgentTool/runAgent.ts#L748)) — the harness recursing on itself with filtered tools and its own `agentId`/transcript (`agent-<id>.jsonl`). Longer-lived or parallel work is modeled as tasks in [tasks/](cc-code/restored-src/src/tasks/) (`LocalAgentTask`, `LocalShellTask`, `RemoteAgentTask`, `InProcessTeammateTask`, `DreamTask`), tracked in `AppState.tasks`. [coordinator/coordinatorMode.ts](cc-code/restored-src/src/coordinator/coordinatorMode.ts) is the multi-agent orchestration mode (leader + workers, `getCoordinatorSystemPrompt`).

**Reference.** `runAgent` [runAgent.ts:248](cc-code/restored-src/src/tools/AgentTool/runAgent.ts#L248), [tools/AgentTool/](cc-code/restored-src/src/tools/AgentTool/), [tasks/](cc-code/restored-src/src/tasks/), [coordinator/coordinatorMode.ts](cc-code/restored-src/src/coordinator/coordinatorMode.ts).

---

## 13. Skills / plugins

**Theory.** *Skills* are packaged, model-invokable procedures (a prompt/instructions + optional allowed-tools + metadata) discovered on demand. *Plugins* are user-installable bundles that contribute commands, skills, MCP servers, and hooks. Both extend the harness without code changes.

**Implementation.** Skills: [skills/loadSkillsDir.ts](cc-code/restored-src/src/skills/loadSkillsDir.ts) loads skill directories; [skills/bundled/](cc-code/restored-src/src/skills/bundled/) ships built-ins (debug, simplify, verify, keybindings…); they surface both as slash commands and via the `SkillTool` ([tools/SkillTool/](cc-code/restored-src/src/tools/SkillTool/)), with optional discovery/prefetch during a turn. Plugins: [plugins/builtinPlugins.ts](cc-code/restored-src/src/plugins/builtinPlugins.ts) + [utils/plugins/pluginLoader.ts](cc-code/restored-src/src/utils/plugins/) load and reconcile plugin state into `AppState.plugins` (enabled/disabled/commands/errors); the loop calls `loadAllPluginsCacheOnly()` at startup so headless never blocks on network. The `/plugin` command manages them.

**Reference.** [skills/loadSkillsDir.ts](cc-code/restored-src/src/skills/loadSkillsDir.ts), [skills/bundled/](cc-code/restored-src/src/skills/bundled/), [tools/SkillTool/](cc-code/restored-src/src/tools/SkillTool/), [plugins/builtinPlugins.ts](cc-code/restored-src/src/plugins/builtinPlugins.ts), [utils/plugins/](cc-code/restored-src/src/utils/plugins/).

---

## 14. Remote

**Theory.** To drive a local session from elsewhere (a phone, claude.ai, a cloud runner), the harness needs a transport that forwards its event stream out and accepts prompts/permission decisions in — bridging the in-process store to a remote client, plus session lifecycle/auth.

**Implementation.** [bridge/](cc-code/restored-src/src/bridge/) is the "always-on bridge" to claude.ai: `bridgeMain.ts` (the `remote-control` subcommand entry), `replBridge.ts`/`initReplBridge.ts` (wire the REPL's event stream and permission callbacks over the bridge), `createSession.ts`, `bridgeMessaging.ts`, JWT/device trust. The corresponding `AppState.replBridge*` fields (dozens of them in [AppStateStore.ts](cc-code/restored-src/src/state/AppStateStore.ts)) reflect connection status into the footer UI. [remote/](cc-code/restored-src/src/remote/) and [cli/transports/](cc-code/restored-src/src/cli/transports/) (`SSETransport`, `WebSocketTransport`, `HybridTransport`, uploaders) carry the actual event traffic for `--remote`/CCR sessions; permission requests can also race over channels (Telegram/iMessage) via `channelPermissionCallbacks`.

**Reference.** [bridge/bridgeMain.ts](cc-code/restored-src/src/bridge/bridgeMain.ts), [bridge/replBridge.ts](cc-code/restored-src/src/bridge/replBridge.ts), [remote/](cc-code/restored-src/src/remote/), [cli/transports/](cc-code/restored-src/src/cli/transports/).

---

*(The table's last row, **SDK**, [entrypoints/sdk/](cc-code/restored-src/src/entrypoints/sdk/): the programmatic surface — `coreTypes.ts`/`coreSchemas.ts` define `SDKMessage`, `HOOK_EVENTS`, and control schemas that `QueryEngine` emits/consumes, so external code can embed the same loop.)*

Want me to now trace one concrete end-to-end path through these layers — e.g. you type a prompt that triggers a Bash call — showing exactly which functions fire in order across rows 5 → 1 → 2 → 8 → 3 → 6?
