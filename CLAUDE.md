# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is an archive of Anthropic's Claude Code CLI , not the live Anthropic
repo. `src/` (~1,900 files, ~512K lines) is the archived leaked TypeScript
source, preserved as a read-only reference. There is no build/test workflow for
`src/` itself — this repo's own development activity is documentation
(`docs/`), the exploration MCP server (`mcp-server/`), and tooling around
studying the code.

**Do not modify `src/`.** It's preserved as-is; the unmodified original is also on the `backup`
branch. Contributions here are documentation, MCP server code, analysis, and exploration tooling.

## Recording what we learn about the shipped binary

`src/` is an **older snapshot** than whatever `claude` is installed. When they disagree,
the binary wins. So we keep a running log of binary-derived findings in
[`how-claude-works.md`](how-claude-works.md), sectioned by version.

**Whenever you inspect the `claude` binary and learn something, append it to
`how-claude-works.md` before you finish the turn.** This applies to any of:

- reading `claude --help` or a subcommand's `--help`
- running `strings` on `~/.local/share/claude/versions/<version>`
- probing behavior empirically (triggering an error path, checking a flag parses)
- finding that `src/` describes something the installed binary no longer does

How to write it:

- Put findings under a `## <version>` heading, e.g. `## 2.1.237`. Get the version and
  commit from `claude doctor`. **Never rewrite an older version's section** — append a new
  one, so behavior changes stay visible over time.
- Tag every claim with its provenance marker: 🔬 binary, 🧪 empirical run, 📦 `src/`
  snapshot, 📖 docs or `CHANGELOG.md`. The file's header defines these. A `src/`-only claim
  that has not been checked against the binary must say so.
- Quote exact strings — flag names, error text, env vars, constants. That is the part that
  is expensive to rediscover.
- Record the *why* when the source explains it. A comment that justifies a magic number is
  worth more than the number.
- **Never quote minified code as-is.** The shipped binary is minified, so anything you pull
  out of it with `strings` has single-letter identifiers. Un-minify it before you write it
  down: give every identifier a descriptive name, add comments explaining what each step
  does and why, and keep the original single-letter name in a trailing `// was: x` comment
  so the mapping stays checkable. Say explicitly that it is un-minified. If the raw form is
  worth keeping, put it in a collapsed `<details>` block underneath, never as the primary
  quote. Minified code is unreadable six months later, which defeats the point of this file.

Do not add findings that came only from the docs; those are already published and change
without notice. This file is for what we had to dig out.

At the end of the conversation, you may also want to update `~/ai-conversations/claude-learning/`.

## Commands

Repo root (checks against the leaked `src/`):
```bash
npm run lint        # Biome lint (biome check src/)
npm run typecheck   # tsc --noEmit
npm run check       # both of the above
```

`mcp-server/` (the actual thing developed in this repo):
```bash
cd mcp-server
npm install
npm run dev      # run with tsx, no build step
npm run build    # compile to dist/
npm run start    # run compiled server (stdio)
npm run start:http  # run compiled server (HTTP)
```

## Architecture

### Exploring `src/`

Start from `Skill.md`, `docs/architecture.md`, `docs/tools.md`,
`docs/commands.md`, `docs/subsystems.md`, and `docs/exploration-guide.md`
— these are curated guides to the leaked codebase and are more reliable/current
than re-deriving structure from scratch. Key landmarks inside `src/`:

- `src/main.tsx` — CLI entrypoint: Commander.js parser + React/Ink renderer. Parallelizes MDM,
  keychain, and GrowthBook fetches on startup before heavy module evaluation.
- `src/QueryEngine.ts` (~46K lines) — core LLM API engine: streaming, tool-use loops, thinking
  mode, retries, token counting.
- `src/Tool.ts` (~29K lines) — base types/interfaces every tool implements (input schemas,
  permissions, progress state).
- `src/commands.ts` (~25K lines) — command registry with conditional per-environment imports.
- `src/tools/` — ~40 self-contained tool implementations, one directory per tool
  (`{ToolName}.ts`/`.tsx` for logic, `UI.tsx` for terminal rendering, `prompt.ts` for the
  system-prompt contribution).
- `src/commands/` — ~85 slash command implementations.
- `src/hooks/toolPermission/` — permission checks on every tool invocation (modes: `default`,
  `plan`, `bypassPermissions`, `auto`, etc.).
- `src/bridge/` — **Remote Control**: the bridge that lets claude.ai/code, the mobile app, and
  Desktop drive a session running on this machine. *Not* the IDE protocol — nothing under
  `bridge/` mentions VS Code or JetBrains. Key files: `bridgeMain.ts` (main loop),
  `remoteBridgeCore.ts`, `createSession.ts`, `sessionRunner.ts` (spawns one child
  `claude --print --sdk-url …` per remote session), `bridgePointer.ts` (the 4-hour
  crash-recovery pointer behind `remote-control --continue`), `workSecret.ts`/`jwtUtils.ts`
  (auth and the `session_ingress` websocket URL), `sessionIdCompat.ts` (`cse_*` ↔ `session_*`
  retagging), `bridgeMessaging.ts` + `inboundMessages.ts` (protocol),
  `bridgePermissionCallbacks.ts`, `replBridge*.ts` (bridging an interactive REPL session).
  See [`how-claude-works.md`](how-claude-works.md) for verified details.
- IDE integration is *not* in `src/bridge/`: see `src/commands/ide/` for the `/ide` command and
  `src/services/mcp/` for the MCP plumbing the VS Code and JetBrains extensions connect over.
- `src/coordinator/` — multi-agent orchestration (sub-agents spawn via `AgentTool`).
- `src/services/` — external integrations: `api/` (Anthropic SDK client), `mcp/`, `oauth/`,
  `lsp/`, `analytics/` (GrowthBook feature flags), `plugins/`, `compact/` (context compression).
- `src/skills/`, `src/plugins/`, `src/memdir/`, `src/tasks/`, `src/state/` — skill system, plugin
  system, persistent memory, task management, state management, respectively.

Request flow: `main.tsx` → `replLauncher.tsx` (REPL session start) → `QueryEngine.ts` (core loop)
→ `services/api/` (Anthropic API call) → tool-use response → `tools/{ToolName}/` (execution) →
result fed back into the loop.

Tools follow a `buildTool()` factory pattern (name, `inputSchema` via Zod, `call()`,
`checkPermissions()`). Feature flags use Bun's `bun:bundle` `feature()` for build-time dead code
elimination (notable flags: `PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`,
`AGENT_TRIGGERS`, `MONITOR_TOOL`).

You must read `Skill.md` now.

### `mcp-server/`

An MCP server (published to npm as `warrioraashuu-codemaster`) that exposes tools for exploring
`src/` from any MCP client: `list_tools`, `list_commands`, `get_tool_source`,
`get_command_source`, `read_source_file`, `search_source`, `list_directory`, `get_architecture`.
It reads the source tree via the `CLAUDE_CODE_SRC_ROOT` env var. Supports stdio, HTTP, and SSE
transports (`src/index.ts` for stdio/build entry, `src/http.js` for HTTP).
