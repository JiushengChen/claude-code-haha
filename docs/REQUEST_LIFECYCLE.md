# Request Lifecycle

How a user prompt travels from keystrokes in the terminal to a final assistant message — and everything that happens in between (slash commands, tool calls, streaming, permissions, abort, retry, compaction).

All file references point at this repo.

---

## High-level flow

```
PromptInput  ─►  REPL.onSubmit  ─►  handlePromptSubmit  ─►  processUserInput
                                                                  │
                                                                  ▼
                                                       QueryEngine.submitMessage
                                                                  │
                                                                  ▼
                                                           query() / queryLoop()
                                                                  │
                                          ┌───────────────────────┼───────────────────────┐
                                          ▼                       ▼                       ▼
                                  callModel (stream SSE)   tool_use detected      no tools → exit
                                          │                       │
                                          │                       ▼
                                          │                runTools (perm check)
                                          │                       │
                                          │                       ▼
                                          │              tool_result messages
                                          │                       │
                                          └──────────► append to state ─► next loop turn
```

Each yielded event flows back through [REPL.tsx:2584](src/screens/REPL.tsx#L2584) into `setMessages`, which re-renders [Messages.tsx](src/components/Messages.tsx) incrementally.

---

## Phase 1 — Input capture

### 1. `PromptInput` submit handler
[src/components/PromptInput/PromptInput.tsx:984-1105](src/components/PromptInput/PromptInput.tsx#L984-L1105)

The TextInput's `onSubmit` validates the buffer (non-empty, not a suggestion-in-progress), resolves `@agent` mentions, attaches pending images/pasted-content references, then calls the parent's `onSubmitProp` with the sanitized text plus helpers.

### 2. REPL routing
[src/screens/REPL.tsx:3142-3520](src/screens/REPL.tsx#L3142-L3520)

`REPL.onSubmit` branches on input shape:

| Branch | Behavior |
|---|---|
| Immediate local-jsx command (e.g. `/config` while a query is running) | Executes in-place, clears input, returns early ([REPL.tsx:3161-3281](src/screens/REPL.tsx#L3161-L3281)) |
| Speculation acceptance | `handleSpeculationAccept` then optional `onQuery` ([REPL.tsx:3392-3406](src/screens/REPL.tsx#L3392-L3406)) |
| Remote mode | Forward over WebSocket ([REPL.tsx:3408-3486](src/screens/REPL.tsx#L3408-L3486)) |
| Normal | Delegates to `handlePromptSubmit` ([REPL.tsx:3490-3519](src/screens/REPL.tsx#L3490-L3519)) |

Also: stash restore, history push, idle-return dialog.

### 3. `handlePromptSubmit` → `executeUserInput`
[src/utils/handlePromptSubmit.ts:120-596](src/utils/handlePromptSubmit.ts#L120-L596)

- Expands pasted-content references.
- Handles `/exit` variants and queued-command bypass.
- Reserves the `queryGuard` (mutex) and creates a fresh `AbortController`.
- Loops queued user inputs, calling `processUserInput` for each.

---

## Phase 2 — Input classification

### 4. `processUserInput`
[src/utils/processUserInput/processUserInput.ts:85-539](src/utils/processUserInput/processUserInput.ts#L85-L539)

Single dispatcher that decides what the input *is*:

| Prefix / mode | Handler | Notes |
|---|---|---|
| `/...` (prompt mode) | `processSlashCommand` ([processUserInput.ts:480](src/utils/processUserInput/processUserInput.ts#L480)) | Skill invocation, local-jsx, attachment extraction |
| `!...` (bash mode) | `processBashCommand` | Runs a shell command, returns output as a user message |
| anything else | `processTextPrompt` | Builds a plain `UserMessage` |

Returns `{ messages, shouldQuery, allowedTools, model, effort }`. If `shouldQuery === true`, the caller invokes the model; otherwise the slash command rendered its own output and we're done.

`UserPromptSubmit` hooks fire here ([processUserInput.ts:182-234](src/utils/processUserInput/processUserInput.ts#L182-L234)) and can inject extra context or veto the turn.

---

## Phase 3 — Query setup

### 5. `QueryEngine.submitMessage`
[src/QueryEngine.ts:209-686](src/QueryEngine.ts#L209-L686)

Assembles everything the model needs:

1. Builds the initial `ProcessUserInputContext` — commands, tools, system prompt fragments ([QueryEngine.ts:335-395](src/QueryEngine.ts#L335-L395)).
2. Runs `processUserInput` (above) to classify input.
3. Merges `allowedTools` into the `ToolPermissionContext.alwaysAllowRules` ([QueryEngine.ts:434-486](src/QueryEngine.ts#L434-L486)).
4. Re-builds the context with the post-classification state — including any model switch from `/model` ([QueryEngine.ts:488-528](src/QueryEngine.ts#L488-L528)).
5. Loads cached skills + plugins ([QueryEngine.ts:534-537](src/QueryEngine.ts#L534-L537)).
6. Emits the `system_init` message — tool / command / skill / plugin inventory shown in the transcript ([QueryEngine.ts:540-551](src/QueryEngine.ts#L540-L551)).
7. If `shouldQuery === false`, yield results and stop.
8. Otherwise invokes `query()` with assembled messages, system prompt, the `canUseTool` permission wrapper, and the `toolUseContext` ([QueryEngine.ts:675-686](src/QueryEngine.ts#L675-L686)).

---

## Phase 4 — The query loop

### 6. `query()` → `queryLoop()`
[src/query.ts:219-1729](src/query.ts#L219-L1729)

A long-running async generator. Each iteration of the `while (true)` at [query.ts:307](src/query.ts#L307) is **one API turn**; multiple turns make up a single user-visible "response" when tools are involved.

Each turn:

1. **(optional) Auto-compaction** — [query.ts:453-468](src/query.ts#L453-L468). If context is oversized, summarize history into a compact summary message first.
2. **(optional) Cached microcompact** — [query.ts:413-426](src/query.ts#L413-L426). Trims old context to keep the prompt-cache hot.
3. **Stream from the model** — [query.ts:659](src/query.ts#L659):
   ```ts
   for await (const message of deps.callModel({ ... })) { ... }
   ```
   `callModel` is `queryModelWithStreaming` in [services/api/claude.ts:752](src/services/api/claude.ts#L752), wrapped by retry logic in [services/api/withRetry.ts](src/services/api/withRetry.ts).
4. **Yield events as they arrive:**
   - Raw `StreamEvent` per SSE chunk ([claude.ts:2299-2303](src/services/api/claude.ts#L2299-L2303)) — text deltas, tool-input deltas, etc.
   - Assembled `AssistantMessage` on each `content_block_stop` ([claude.ts:2210](src/services/api/claude.ts#L2210)). The final `message_delta` mutates that message in place so usage / stop_reason are captured.
5. **Detect tool use** — [query.ts:829-835](src/query.ts#L829-L835). Any `content.type === 'tool_use'` blocks get pushed onto `toolUseBlocks[]` and the loop sets `needsFollowUp = true`.
6. **(optional) Streaming tool execution** — [query.ts:838-862](src/query.ts#L838-L862). Behind a flag, tools begin running while the model is still emitting later blocks.

### 7. Tool dispatch
[src/utils/toolOrchestration.ts:19-70](src/utils/toolOrchestration.ts#L19-L70), tool execution at [src/utils/toolExecution.ts](src/utils/toolExecution.ts)

For each `tool_use` block:

1. **Permission check** via the `canUseTool` callback passed in from `QueryEngine`. Reads `ToolPermissionContext` (see [utils/permissions/](src/utils/permissions/)). Denied → synthetic error `tool_result`.
2. **Pre/PostToolUse hooks** ([utils/hooks.ts](src/utils/hooks.ts)). Hooks can block, inject context, or modify inputs.
3. **Execution.** Concurrent-safe tools run in parallel; others serially. Each tool returns a user-role message containing a `tool_result` block.
4. **Yield results** as they complete ([query.ts:1380-1408](src/query.ts#L1380-L1408)).

### 8. Next iteration
[query.ts:1715-1727](src/query.ts#L1715-L1727)

```text
new state.messages = [
  ...messagesForQuery,   // prior history
  ...assistantMessages,  // model output from this turn
  ...toolResults,        // user messages with tool_result blocks
]
```

Loop returns to step 6.3 with the augmented history. The next API call sees the tool results and decides whether to call more tools or finish.

---

## Phase 5 — Termination

The loop exits with a `Terminal` object ([query.ts:1062-1357](src/query.ts#L1062-L1357)):

| `reason` | Trigger | Where |
|---|---|---|
| `completed` | No tool_use blocks in the latest assistant turn | [query.ts:1357](src/query.ts#L1357) |
| `max_turns` | Turn counter exhausted | [query.ts:1705-1711](src/query.ts#L1705-L1711) |
| `aborted_streaming` / `aborted_tools` | User Ctrl-C → `AbortController.signal` | [query.ts:1015-1051](src/query.ts#L1015-L1051) |
| `stop_hook_prevented` | Stop hook set `preventContinuation` | [query.ts:1278-1305](src/query.ts#L1278-L1305) |
| `prompt_too_long` / `image_error` | Unrecoverable API error | [query.ts:1175](src/query.ts#L1175), [1182](src/query.ts#L1182), [1254](src/query.ts#L1254) |

---

## Phase 6 — UI updates

The REPL consumes the generator at [REPL.tsx:2793-2802](src/screens/REPL.tsx#L2793-L2802):

```ts
for await (const event of query({ ... })) {
  onQueryEvent(event)
}
```

`onQueryEvent` ([REPL.tsx:2584](src/screens/REPL.tsx#L2584)) routes through `handleMessageFromStream` to `setMessages` ([REPL.tsx:1198](src/screens/REPL.tsx#L1198)), which appends to React state. [Messages.tsx](src/components/Messages.tsx) re-renders incrementally on every yield — that's how you see text streaming in.

---

## Cross-cutting concerns

| Concern | Where |
|---|---|
| Abort | `AbortController` created in `handlePromptSubmit`, signal threaded through `toolUseContext`, checked in [query.ts:664, 1015](src/query.ts#L664) and inside long-running tools |
| Retry | [services/api/withRetry.ts](src/services/api/withRetry.ts); fallback-model swap via `FallbackTriggeredError` at [query.ts:894](src/query.ts#L894) |
| Auto-compaction | [services/compact/](src/services/compact/) invoked from [query.ts:453-468](src/query.ts#L453-L468) |
| Hooks | `UserPromptSubmit` ([processUserInput.ts:182-234](src/utils/processUserInput/processUserInput.ts#L182-L234)), `PreToolUse` / `PostToolUse` / `Stop` ([utils/hooks.ts](src/utils/hooks.ts)) |
| Permission rules | [utils/permissions/](src/utils/permissions/) + bash command classifier |
| Input branches | Slash command, bash `!`, immediate local-jsx, speculation, remote, plain prompt — all in [REPL.tsx:3142-3520](src/screens/REPL.tsx#L3142-L3520) |

---

## TL;DR

A prompt flows: **PromptInput → REPL.onSubmit → handlePromptSubmit → processUserInput** (classification) **→ QueryEngine.submitMessage** (assemble tools/system/skills) **→ query()** (streaming loop: API → detect tool_use → permissions → run tools → append results → repeat) **→ Terminal** (completed / max_turns / aborted / error). Each yielded event drives `setMessages` in the REPL, which re-renders the transcript live.
