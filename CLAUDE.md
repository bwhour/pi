# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Run checks (linting, type checking, import validation) - REQUIRED after code changes
npm run check

# Run all tests (from repo root)
npm run test

# Run a specific test file (from the package root, e.g. packages/coding-agent)
npx tsx ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts

# Build all packages in dependency order
npm run build

# Run pi from source (with your env API keys)
./pi-test.sh

# Run pi from source without any API keys
./pi-test.sh --no-env

# Build full binary (node + bun)
cd packages/coding-agent && npm run build:binary
```

**NEVER run `npm run build` or `npm test` directly** (except as noted above). Always use `npm run check` after code changes, not tests.

**Dependency installs:** Always use `--ignore-scripts` flag:
```bash
npm install --ignore-scripts
npm ci --ignore-scripts
npm install --package-lock-only --ignore-scripts  # metadata-only update
```

## Architecture

This is a monorepo (`pi-monorepo`) with four packages, built in dependency order:

```
packages/tui      → packages/ai → packages/agent → packages/coding-agent
```

### `packages/ai` — LLM provider abstraction layer

Unified streaming API over multiple LLM providers. Key files:
- `src/types.ts` — `Api`, `Provider`, `KnownProvider`, `StreamOptions`, `Model` types
- `src/providers/` — one file per provider (anthropic, google, openai-completions, bedrock, mistral, etc.)
- `src/providers/register-builtins.ts` — lazy-loads providers; never statically import provider modules here
- `src/models.generated.ts` — **NEVER edit directly**; regenerate via `src/scripts/generate-models.ts`
- `src/env-api-keys.ts` — env var → credential detection per provider

All providers implement `stream()` returning `AssistantMessageEventStream` with standardized events (`text`, `tool_call`, `thinking`, `usage`, `stop`).

### `packages/agent` — Provider-agnostic agent loop

Wraps `pi-ai` with tool execution, message history, and transport abstraction:
- `src/agent-loop.ts` — core loop driving tool calls and message accumulation
- `src/agent.ts` — high-level `Agent` class
- `src/types.ts` — `StreamFn`, `ToolExecutionMode`, `QueueMode`, `AgentToolCall`, `BeforeToolCallResult`
- `src/harness/` — test harness with faux provider (use for suite tests, not real APIs)

### `packages/tui` — Terminal UI primitives

Reusable TUI components and stdin handling:
- `src/tui.ts`, `src/terminal.ts` — low-level terminal control
- `src/components/` — composable UI components
- `src/keybindings.ts` — all keybindings must go through `DEFAULT_EDITOR_KEYBINDINGS` or `DEFAULT_APP_KEYBINDINGS`; never hardcode key strings

### `packages/coding-agent` — CLI coding agent (the `pi` binary)

Highest-level package; ships as `pi` CLI:
- `src/cli.ts` — entry point
- `src/core/` — session management, settings, auth, tools, extensions, compaction, slash commands, skills
- `src/core/tools/` — file tools: bash, read, write, edit, find, grep, ls
- `src/core/extensions/` — extension loader/runner (`.pi/extensions/`)
- `src/modes/interactive/` — TUI interactive mode
- `src/modes/print-mode.ts` — non-interactive print mode
- `src/modes/rpc/` — JSON-RPC mode for programmatic use (`rpc-types.ts` defines the protocol)
- `src/cli/` — CLI arg parsing, model listing, session picker, config selector

Config lives in `.pi/` at project root or `~/.pi/` for user-level settings.

## TypeScript Constraints

Use only erasable TypeScript (compatible with Node strip-only mode). **Forbidden constructs:**
- Constructor parameter properties → use explicit fields + constructor assignments
- `enum` → use string literal unions
- `namespace` / `module` / `import =` / `export =`
- Inline/dynamic imports (`await import(...)`, `import("pkg").Type`) → always top-level imports
- `any` types (unless absolutely necessary)
- Single-line helper functions with a single call site → inline them

## Testing

- Suite tests live in `packages/coding-agent/test/suite/` and use `test/suite/harness.ts` + faux provider
- Issue regressions: `packages/coding-agent/test/suite/regressions/<issue-number>-<short-slug>.test.ts`
- If you create or modify a test file, run it and iterate until it passes
- Do not use real provider APIs or paid tokens in tests

## Key Invariants

- **Lockfile commits** are blocked by pre-commit hook; set `PI_ALLOW_LOCKFILE_CHANGE=1` only when intentional
- **Keybindings** must always be configurable via the default keybinding objects; never call `matchesKey(keyData, "ctrl+x")`
- **`models.generated.ts`** is auto-generated; edit `scripts/generate-models.ts` instead
- **Parallel agents** must `git add <specific-file-paths>` only — never `git add -A` or `git add .`
- **Changelog** entries always go under `## [Unreleased]` in `packages/*/CHANGELOG.md`; never modify released sections

## tmux Testing for TUI

```bash
tmux new-session -d -s pi-test -x 80 -y 24
tmux send-keys -t pi-test "cd /path/to/pi-mono && ./pi-test.sh" Enter
sleep 3 && tmux capture-pane -t pi-test -p
tmux send-keys -t pi-test "your prompt" Enter
tmux kill-session -t pi-test
```
