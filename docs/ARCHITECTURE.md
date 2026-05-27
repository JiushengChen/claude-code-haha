# Architecture

## What This Is

A local-runnable fork of the leaked Claude Code source (`version 999.0.0-local`). The original leaked source couldn't run directly; this repo patches the startup chain to make the full Ink TUI and `--print` headless mode work locally. Supports any Anthropic-compatible API backend via `ANTHROPIC_BASE_URL`.

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Bun |
| Language | TypeScript (ESNext, JSX via React) |
| UI | [Ink](https://github.com/vadimdemedes/ink) — React for terminal |
| API client | `@anthropic-ai/sdk` |
| AI providers | Anthropic, AWS Bedrock, GCP Vertex, Azure Foundry |
| MCP | `@modelcontextprotocol/sdk` |
| Observability | OpenTelemetry, GrowthBook feature flags |
| Schema | Zod v4, AJV |
| CLI parsing | `@commander-js/extra-typings` |

## Directory Layout

```
/
├── bin/claude-haha             # Shell entrypoint
├── preload.ts                  # Bun preload — sets MACRO globals (version, build time)
├── bunfig.toml                 # Tells Bun to preload preload.ts
├── stubs/                      # No-op replacements for unavailable native modules
│   ├── color-diff-napi.ts
│   └── ant-claude-for-chrome-mcp.ts
└── src/
    ├── entrypoints/
    │   └── cli.tsx             # Fast-path CLI router (version, daemon, bridge, bg...)
    ├── main.tsx                # Commander setup + REPL launch
    ├── setup.ts                # Per-session init (cwd, hooks, session history)
    ├── bootstrap/state.ts      # Global mutable singleton (session ID, tokens, hooks)
    ├── commands.ts             # Slash command registry
    ├── commands/               # ~50 sub-command implementations
    ├── tools.ts                # getTools() — assembles active tool list
    ├── Tool.ts                 # Tool interface definition
    ├── tools/                  # ~40 tool implementations
    │   ├── BashTool/
    │   ├── FileReadTool/ FileWriteTool/ FileEditTool/
    │   ├── GlobTool/ GrepTool/
    │   ├── AgentTool/          # Spawns sub-agents
    │   ├── WebFetchTool/ WebSearchTool/
    │   ├── MCPTool/
    │   ├── SkillTool/
    │   ├── TodoWriteTool/
    │   └── Task*/              # Task CRUD tools
    ├── query.ts                # Core LLM request loop (streaming, tool dispatch)
    ├── QueryEngine.ts          # Stateful wrapper around query.ts
    ├── context.ts              # System prompt assembly (git status, CLAUDE.md, memory)
    ├── screens/
    │   ├── REPL.tsx            # Main interactive screen
    │   └── Doctor.tsx          # Diagnostics screen
    ├── components/             # ~150 Ink/React UI components
    │   ├── App.tsx             # Top-level provider tree
    │   ├── PromptInput/
    │   ├── Messages.tsx
    │   ├── permissions/        # Permission dialogs
    │   └── diff/               # Diff viewer
    ├── state/
    │   ├── AppStateStore.ts    # AppState type + defaults
    │   ├── AppState.tsx        # React context provider
    │   └── store.ts            # Observable store (useSyncExternalStore)
    ├── services/
    │   ├── api/                # API client — streams from Anthropic-compatible endpoint
    │   ├── mcp/                # MCP connection manager
    │   ├── analytics/          # GrowthBook, event logging
    │   ├── compact/            # Auto-context-compaction
    │   └── SessionMemory/
    ├── utils/
    │   ├── model/              # Model resolution, provider selection, pricing
    │   ├── permissions/        # Rules engine + bash command classifier
    │   ├── settings/           # settings.json parsing, MDM, change detection
    │   ├── git.ts              # Git helpers
    │   ├── auth.ts             # OAuth, API key, Bedrock/Vertex creds
    │   ├── hooks.ts            # Pre/post-tool hooks (bash, HTTP, prompt)
    │   └── swarm/              # Multi-agent coordination
    ├── bridge/                 # WebSocket relay to claude.ai
    ├── coordinator/            # Multi-agent orchestration (coordinator mode)
    ├── assistant/              # KAIROS assistant mode
    ├── skills/                 # User-defined reusable skills
    ├── vim/                    # Vim mode for prompt input
    ├── voice/                  # Voice input
    └── localRecoveryCli.ts     # Fallback readline REPL (no Ink, crash recovery)
```

## Startup Flow

```
bin/claude-haha
  └─ bun --env-file=.env src/entrypoints/cli.tsx
       ├─ preload.ts fires (bunfig.toml) → sets globalThis.MACRO
       ├─ cli.tsx fast-paths: --version, --daemon-worker, remote-control, ps/logs/attach
       └─ main.tsx
            ├─ init(): config, env vars, TLS, telemetry, MDM (all parallel)
            ├─ registers Commander CLI commands
            └─ launchRepl() → renders <App><REPL/></App> via Ink
```

## Request Lifecycle

1. User types in `PromptInput` → `REPL.tsx` submits
2. `QueryEngine.runQuery()` → `query()` in `query.ts`
3. `query.ts` calls `services/api/claude.ts` — streams from Anthropic API
4. Streaming chunks update `AppState` messages → `Messages.tsx` re-renders
5. Tool use blocks dispatched → `Tool.call()` executes → result appended as user message → next API turn

## State Layers

| Store | Contents |
|---|---|
| `bootstrap/state.ts` | Low-level singleton: session ID, cwd, token counters, hooks registry, model override |
| `state/AppStateStore.ts` | React-level: messages, permission mode, MCP connections, settings, task list |
| `state/store.ts` | Generic observable (`useSyncExternalStore`) backing AppState |

## Tool System

- `getTools()` assembles the active list, filtered by feature flags and `USER_TYPE`
- Each tool implements: `name`, `inputSchema`, `call()`, `renderToolUseMessage()`, `renderResultMessage()`, `isEnabled()`, `needsPermissions()`
- Permissions checked via `CanUseToolFn` → rules engine in `utils/permissions/`

## AI Provider Routing

| Env var | Provider |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK=1` | AWS Bedrock |
| `CLAUDE_CODE_USE_VERTEX=1` | GCP Vertex |
| `CLAUDE_CODE_USE_FOUNDRY=1` | Azure Foundry |
| `ANTHROPIC_BASE_URL=<url>` | Custom proxy / compatible API |
| _(default)_ | Anthropic direct |

## Config Hierarchy

1. Global `~/.claude/settings.json`
2. Per-project `.claude/settings.json`
3. MDM-managed settings
4. Remote-managed settings (fetched at startup)

All merged in `utils/settings/settings.ts`.

## Permission Modes

`default` → `acceptEdits` → `dontAsk` → `bypassPermissions` → `plan` → `auto`

Enforced by the rules engine (`utils/permissions/`) and a bash command classifier.

## Multi-Agent (Swarm)

`AgentTool` spawns sub-agents. `utils/swarm/` handles leader/worker coordination, permission sync via mailbox, and teammate context sharing.

## Notable Patterns

- **Build-time DCE** — `bun:bundle` `feature()` calls strip internal features (`COORDINATORMODE`, `KAIROS`, `VOICE_MODE`, `DAEMON`, `BRIDGE_MODE`) from external builds.
- **Startup performance** — keychain prefetch, MDM raw read, and API pre-connect fire in parallel before the UI mounts; `startupProfiler.ts` checkpoints each phase.
- **Stubs** — `stubs/` replaces native binaries with no-op TypeScript, mapped via `tsconfig.json` paths.
- **Migrations** — `src/migrations/` handles model name changes and settings schema evolution automatically on startup.
