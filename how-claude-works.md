# How `claude` works — findings from the binary

A running log of what we learn about the **shipped `claude` binary**, as opposed to
the archived `src/` in this repo. `src/` is an older snapshot; the binary is what
actually runs. When the two disagree, the binary wins.

Add a new `##` section per version. Never rewrite an old version's section — append a
new one, so we can see behavior change over time.

## Provenance markers

Every claim carries one, because they are not equally trustworthy:

| Marker | Means |
|---|---|
| 🔬 | Verified against the installed binary: `--help`, `strings`, or an actual run |
| 🧪 | Observed empirically — a command was run and this was the output |
| 📦 | Read from this repo's `src/` snapshot. Mechanism only; may be stale |
| 📖 | From <https://code.claude.com/docs> or `CHANGELOG.md`, not the binary |

---

## 2.1.234

Recorded **2026-08-20**, but the work was done in a session running **2.1.234**, before this
machine auto-updated to 2.1.237. The 📦 claims come from this repo's `src/` snapshot and are
not version-bound; the 🧪 run below was made on 2.1.234.

### Memory files: the user rules directory

📦 `src/utils/claudemd.ts:1` documents the load order. Later files win, so the model weighs
them more:

1. Managed — `/etc/claude-code/CLAUDE.md` + `<managed>/.claude/rules/*.md`
2. User — `~/.claude/CLAUDE.md` + `~/.claude/rules/**/*.md`
3. Project — `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`, walked cwd → root
4. Local — `CLAUDE.local.md`

📦 `src/utils/config.ts:1805`:

```ts
export function getUserClaudeRulesDir(): string {
  return join(getClaudeConfigHomeDir(), 'rules')
}
```

⇒ `~/.claude/rules/` is a **user-level, Claude-Code-only** instruction channel, and it needs
no `settings.json` entry — the directory is auto-discovered. This is the answer to "how do I
give Claude Code instructions that the other agents sharing my `CLAUDE.md` never see".

📦 `processMdRules()` (`claudemd.ts:697`) recurses into subdirectories, accepts `.md` only,
resolves symlinks with cycle detection (`visitedDirs`), and swallows `ENOENT`/`EACCES`/`ENOTDIR`.
It runs under `isSettingSourceEnabled('userSettings')` with `includeExternal: true`, because
"User memory can always include external files".

🧪 Verified on 2.1.234: a file dropped into `~/.claude/rules/` appears in the system prompt of
an unrelated `claude --print` run started in `/tmp`.

#### The conditional-rule frontmatter key is `paths:`, not `globs:`

📦 `parseFrontmatterPaths()` (`claudemd.ts:254`) reads `frontmatter.paths`. The parsed value is
only *stored* on an internal field named `globs`, which is what makes this easy to get wrong.

- A trailing `/**` is stripped, because the `ignore` library already treats `path` as matching
  the directory and everything inside it.
- If every pattern is `**`, the globs are dropped and the file becomes unconditional.
- `conditionalRule` splits rules into two disjoint sets: `files.filter(f => conditionalRule ? f.globs : !f.globs)`.
  No `paths:` ⇒ always in context. With `paths:` ⇒ loaded only when a matching file is reached.

⚠️ 📦 **`AGENTS.md` is never auto-loaded.** `rg 'AGENTS\.md' src/` hits only
`src/commands/init.ts`, where `/init` *reads* it to help write `CLAUDE.md`. Nothing in the
memory loader looks for it.

### SessionStart hooks

📦 Four triggers, taken from the `processSessionStartHooks(...)` call sites:

| Trigger | Fired from |
|---|---|
| `startup` | `main.tsx`, `cli/print.ts` |
| `resume` | `screens/REPL.tsx`, `utils/conversationRecovery.ts` |
| `clear` | `commands/clear/conversation.ts` |
| `compact` | `services/compact/compact.ts`, `services/compact/sessionMemoryCompact.ts` |

⇒ Match `startup` explicitly, or the hook also re-runs after every compaction.

📦 A SessionStart hook's `additionalContext` is injected as a `hook_additional_context`
attachment message (`utils/sessionStart.ts:163`), so hook stdout costs tokens in **every**
session. 📦 `main.tsx:3762` notes that startup avoids "blocking ~500ms waiting for SessionStart
hooks to finish", so a slow hook is felt at launch.

📦 `Setup` hooks are **not** scheduled maintenance: `processSetupHooks(trigger: 'init' | 'maintenance')`
runs only behind the `--init`, `--init-only`, or `--maintenance` flags.

---

## 2.1.237

Recorded **2026-08-20**. Verified on `darwin-arm64` (Mac) and `linux-x64` (veLinux devbox).

```
Running: native (2.1.237)
Commit:  45590910f0b7
Path:    ~/.local/share/claude/versions/2.1.237
Size:    317,110,288 bytes (Mach-O 64-bit executable arm64)
```

🧪 `claude doctor` reports: `Running`, `Commit`, `Platform`, `Path`,
`Config install method`, `Search`, `Auto-updates`, `Auto-update channel`,
`Last update attempt`.

🧪 `claude auth status` prints JSON, not prose:
`{loggedIn, authMethod, apiProvider, email, orgId, orgName, subscriptionType}`.
Useful for scripting an entitlement check.

### Subcommands

🔬 `claude --help` lists only these under `Commands:`

```
agents  auth  auto-mode  doctor  gateway  import  install  mcp
plugin|plugins  project  setup-token  ultrareview  update|upgrade
```

⚠️ **`remote-control` and `self-hosted-runner` are NOT in that list**, yet
`claude remote-control --help` works. They are hidden or entitlement-gated.
🧪 `claude self-hosted-runner --help` fell through to the general help on a personal
Max account — consistent with it being Team/Enterprise-only.

### Hidden and gated flags

📦 From `main.tsx`:

- `--advisor <model>` — added with `.hideHelp()`, gated behind `canUserConfigureAdvisor()`
- `--delegate-permissions` — `[ANT-ONLY] Alias for --permission-mode auto`, gated by a
  build-time `if ("external" === 'ant')` check. The string literal is how the external
  build dead-codes it out.

🔬 `--rc` is accepted as an alias for `--remote-control` but is **not printed in `--help`**.
🧪 `claude --rc --help` parses without error.

### Remote Control

🔬 `claude remote-control --help` (eligibility is checked *before* help prints, so this
errors instead of showing flags when signed out):

| Flag | Notes |
|---|---|
| `--name <name>` | Session title shown at claude.ai/code |
| `--remote-control-session-name-prefix <prefix>` | Default: hostname. Env: `CLAUDE_REMOTE_CONTROL_SESSION_NAME_PREFIX` |
| `-c, --continue` | Reattach to the session last recorded **for this directory** (or one of its git worktrees). Errors if nothing recorded in ~4h |
| `--session-id <id>` | Reattach one session by ID |
| `--permission-mode <mode>` | `acceptEdits, auto, bypassPermissions, default, dontAsk, plan` |
| `--spawn <mode>` | `same-dir` (default), `worktree`, `session` |
| `--capacity <N>` | Default 32. Cannot combine with `--spawn=session` |
| `--[no-]create-session-in-dir` | Default on |
| `--debug-file <path>`, `-v/--verbose` | |

🔬 Binary string: `- Worktree mode requires a git repository or WorktreeCreate/WorktreeRemove hooks`

Related top-level flags 🔬: `--remote-control [name]`, `--teleport [session]`,
`--cloud [description|session_id|url]`, `--environment <environment_id>`.

#### The bridge pointer — why `--continue` expires after 4 hours

This is the most useful thing we worked out. 📦 `src/bridge/bridgePointer.ts`:

```ts
export const BRIDGE_POINTER_TTL_MS = 4 * 60 * 60 * 1000
const MAX_WORKTREE_FANOUT = 50
```

- Path: `~/.claude/projects/<sanitized-cwd>/bridge-pointer.json`
- Schema: `{ sessionId, environmentId, source: 'standalone' | 'repl' }`
- Written when a bridge session is created, **periodically re-written with identical
  content**, and cleared on clean shutdown.
- Staleness is measured by the file's **mtime**, not an embedded timestamp. That is
  deliberate: a no-op rewrite counts as a refresh. The source comment says it
  "matches the backend's rolling `BRIDGE_LAST_POLL_TTL` (4h) semantics. A bridge that's
  been polling for 5+ hours and then crashes still has a fresh pointer as long as the
  refresh ran within the window."
- ⇒ **The 4-hour window is rolling, not absolute.** It starts when the process stops
  refreshing, not when the session began. A server left running never expires.
- ⇒ The client TTL exists to avoid promising a resume the *server* can no longer honor.
- Scoped per working directory so two bridges in different repos never clobber each other.
- If the current directory has no pointer, `--continue` fans out across the repo's git
  worktrees, capped at `MAX_WORKTREE_FANOUT = 50`; above that it falls back to
  current-dir-only.

🔬 Confirmed present in the 2.1.237 binary via `strings`:

```
bridge-pointer.json
[bridge:pointer] wrote
[bridge:pointer] cleared
[bridge:pointer] clear failed:
[bridge:pointer] write failed:
[bridge:pointer] invalid schema, clearing:
[bridge:pointer] stale (>4h mtime), clearing:
[bridge:pointer] fanout found pointer in worktree
```

#### How server mode runs sessions

📦 `src/bridge/sessionRunner.ts` — the server **spawns child `claude` processes**:

```
--print --sdk-url <url> --session-id <id>
--input-format stream-json --output-format stream-json
```

`scriptArgs` is prepended for npm installs (where `execPath` is node and `process.argv[1]`
is the script), and empty for compiled binaries. Without it, node parses `--sdk-url` as its
own option and dies with `bad option: --sdk-url` — anthropics/claude-code#28334.

⇒ **Consequence worth remembering:** server-mode sessions are `--print` sessions, so
📖 they do **not** appear in the `claude --resume` picker. You must pass the session ID.

#### Two kinds of bridge session — and why a restarted server cannot see the other kind

Found 2026-09-02 on the dev box, Claude Code 2.1.258. A session that was "online" at
claude.ai/code went to **Remote Control disconnected** and a freshly started
`claude remote-control` in the same directory did not bring it back.

There are two producers of bridge sessions, and each registers its **own environment**:

| Kind | Process | Transcript `entrypoint` | Pointer `source` | Environment |
|---|---|---|---|---|
| REPL bridge | interactive `claude` + `/remote-control`, `--rc`, or `remoteControlAtStartup` | `cli` | `repl` | one per REPL process (`src/bridge/replBridge.ts` registers with a fresh `randomUUID()`) |
| Standalone server | `claude remote-control` | children are `sdk-cli` | `standalone` | one per server, printed in the banner URL as `?environment=env_…` |

- A restarted standalone server only re-adopts sessions in **its** environment. A REPL
  bridge session lives in the REPL's environment, so the server never sees it. The docs
  say the same in one line: "To bring back a session you started with
  `claude --remote-control` or `/remote-control`, resume the conversation with
  `claude --continue` or `claude --resume`."
- `replBridge.ts:312` also refuses a `standalone` pointer for perpetual REPL resume, and
  `--continue` in `bridgeMain.ts` chains into the `--session-id` flow, which is
  single-session and rejects `--spawn`. So a wrapper that always passes `--spawn=…`
  (like `cl`) can never be the thing that resumes.
- 🔬 The transcript records the link as `{"type":"bridge-session","bridgeSessionId":"cse_…"}`
  lines. `claude --resume <uuid>` reads those and reconnects (docs: "reconnection record").
- 🔬 `~/.claude.json` holds `replBridgePlaceholders.<cse_id> = {pid, procStart, createdAt}`
  while a REPL bridge is alive; it is removed on clean exit.
- ⚠️ `remoteControlAtStartup: true` in `~/.claude/settings.json` means a bare `claude` in a
  terminal silently becomes a REPL bridge. Its claude.ai row looks identical to a
  server-spawned one; the `entrypoint` field in the JSONL is the reliable tell.

#### Session ID retagging

📦 `src/bridge/sessionIdCompat.ts` — the same UUID wears two tags:

- Worker endpoints `/v1/code/sessions/{id}/worker/*` want `cse_*`
- Client compat endpoints `/v1/sessions/{id}`, `/archive`, `/events` want `session_*`

`toCompatSessionId()` re-tags `cse_` → `session_`, gated by a GrowthBook kill switch
injected via `setCseShimGate()` to avoid a static import chain banned from the SDK bundle.

📦 `src/bridge/workSecret.ts`: `buildSdkUrl()` →
`{ws,wss}://<host>/<version>/session_ingress/ws/<sessionId>`.

✅ **Fixed in this repo's `CLAUDE.md` on 2026-08-20.** It used to describe `src/bridge/`
as the "IDE integration protocol (VS Code, JetBrains)". That was wrong: 🧪 `rg -i
'vscode|jetbrains|intellij' src/bridge/` returns **zero** matches. `src/bridge/` is
**Remote Control** — the claude.ai / mobile / Desktop bridge. Real IDE integration lives in
`src/commands/ide/` and `src/services/mcp/`.

### `--worktree` and `--tmux`

🔬 Help text:

```
-w, --worktree [name]   Create a new git worktree for this session (optionally specify a name)
--tmux                  Create a tmux session for the worktree (requires --worktree).
                        Uses iTerm2 native panes when available; use --tmux=classic
                        for traditional tmux.
```

⚠️ `--tmux` is **undocumented** on <https://code.claude.com/docs/en/worktrees>. Help text
and the binary are the only sources. Earliest CHANGELOG mention is a 2.1.157 bugfix, so it
predates that.

📦 `src/utils/worktree.ts` → `execIntoTmuxWorktree()`, invoked from `entrypoints/cli.tsx:247`
as a **fast path before the full CLI loads**.

Mechanism 📦:

1. Refuse on Windows; refuse if `tmux -V` fails.
2. Parse the worktree name and `--tmux=classic` straight out of `argv`.
3. Resolve a PR reference (`#1234` or a PR/MR URL) to `pr-<n>`.
4. 🔬 Generate a slug if unnamed — the fast path has its **own** word lists (this
   literal array is present in the shipped binary, not just `src/`):
   `[swift, bright, calm, keen, bold]` × `[fox, owl, elm, oak, ray]` + a 4-char base36
   suffix. (Different from the general `--worktree` generator, which produces things like
   `bright-running-fox`.)
5. `validateWorktreeSlug()`.
6. A `WorktreeCreate` hook takes precedence over git — anthropics/claude-code#39281.
7. tmux session name = `` `${repoName}_${worktreeBranchName(name)}` `` with `/` and `.`
   replaced by `_`. So repo `myapp` + `-w feature-auth` → `myapp_worktree-feature-auth`.
8. Strip `--tmux`, `--tmux=classic`, `--worktree` and its value from argv; forward the rest
   to the inner `claude`.

Slug rules 📦:

```
MAX_WORKTREE_SLUG_LENGTH = 64
```

Split on `/`; each segment must be non-empty and match an allowlist of letters, digits,
dots, underscores, dashes. `.` and `..` segments are rejected. Slashes are allowed so
`asm/feature-foo` nests. The comment explains why: the slug is `path.join`ed into
`.claude/worktrees/<slug>`, which normalizes `..`, so `../../../target` would escape.

#### Control mode vs classic

🔬 *Un-minified* from the binary's `E=eRe()&&!n&&!v`:

```js
const useControlMode =
  isInITerm2() &&        // eRe()
  !forceClassicTmux &&   // !n — n is set by --tmux=classic
  !isAlreadyInTmux       // !v — v = Boolean(process.env.TMUX)
```

Control mode passes `-CC` so iTerm2 renders native tabs and panes.

🔬 The iTerm2 hint string is in the binary:
`iTerm2 > Settings > General > tmux > "Tabs in attaching window"`

#### Prefix conflict detection

🔬 Reads `tmux show-options -g prefix`, then compares against Claude's own bindings
(the array is verbatim in the binary):

```
C-b  C-c  C-d  C-t  C-o  C-r  C-s  C-g  C-e
```

⚠️ **The default tmux prefix `C-b` collides** with Claude's `ctrl+b` (task:background).

Env vars passed to the inner Claude 🔬 (all four confirmed in the 2.1.237 binary):

```
CLAUDE_CODE_TMUX_SESSION
CLAUDE_CODE_TMUX_PREFIX
CLAUDE_CODE_TMUX_PREFIX_CONFLICTS
CLAUDE_CODE_TMUX_TRUECOLOR     ← in the binary, absent from the src/ snapshot
```

#### Binary-only: the tmux version gate 🔬

**Not in the `src/` snapshot.** 2.1.237 parses the version out of `tmux -V` and chooses
how to hand the env vars to the child:

*Un-minified.* The binary ships single-letter identifiers; names below are inferred
from usage, and the original is noted per line.

```js
// `tmuxVersionResult` (t) is the earlier `await run("tmux", ["-V"])`.
// `tmuxEnv` (f) is { CLAUDE_CODE_TMUX_SESSION, CLAUDE_CODE_TMUX_PREFIX,
//                    CLAUDE_CODE_TMUX_PREFIX_CONFLICTS }.

const tmuxVersionMatch = tmuxVersionResult.stdout.match(/(\d+)\.(\d+)/)
const major = Number(tmuxVersionMatch?.[1])
const minor = Number(tmuxVersionMatch?.[2])

// tmux gained `-e KEY=VALUE` on new-session in 3.2, so gate on >= 3.2.
const supportsEnvFlag =
  tmuxVersionMatch !== null && (major > 3 || (major === 3 && minor >= 2))

// Modern tmux: hand the vars over as explicit -e arguments.
const envFlagArgs = supportsEnvFlag
  ? Object.entries(tmuxEnv).flatMap(([key, value]) => ["-e", `${key}=${value}`])
  : []

// Old tmux: fall back to leaking them through the child's environment.
const childEnv = { ...process.env, ...(supportsEnvFlag ? {} : tmuxEnv) }
```

<details><summary>Original minified form, as it appears in the binary</summary>

```js
m=t.stdout.match(/(\d+)\.(\d+)/),h=m!==null&&(Number(m[1])>3||Number(m[1])===3&&Number(m[2])>=2),g=h?Object.entries(f).flatMap(([P,I])=>["-e",`${P}=${I}`]):[],y={...process.env,...h?{}:f},
```

</details>

- **tmux ≥ 3.2** → pass them as `tmux -e KEY=VALUE` arguments.
- **older tmux** → fall back to merging them into the child's `process.env`.

`-e` on `new-session` landed in tmux 3.2, so this is a graceful degradation. The devbox
runs tmux 3.3a, so it takes the `-e` path.

#### Binary-only: hook path hardening 🔬

Also absent from the snapshot. When a `WorktreeCreate` hook supplies the path, 2.1.237
screens it twice before using it:

```
the WorktreeCreate hook emitted a path with dot segments (
the WorktreeCreate hook's worktree path component
```

The first rejects dot segments, because "the symlink screen cannot verify a dotted
spelling". The second walks each path component looking for a symlink, because "a
repository-committed symlink below the checkout root could redirect the worktree outside
the repository".

🔬 Order of checks also differs from the snapshot: **workspace trust is checked first**,
before `tmux -V`. So an untrusted directory reports the trust error even with tmux missing.

#### Nesting

📦 When already inside tmux it creates a **detached sibling** session and runs
`switch-client`, rather than nesting. If the session already exists it only switches.

#### Ant-only easter egg

📦 When `USER_TYPE=ant` **and** the repo is named `claude-cli-internal`, the fast path
builds a three-pane layout: Claude in pane 0, `bun run watch` split horizontally,
`bun run start` split vertically.

#### Verified error paths

🧪 Run against 2.1.237:

| Condition | Message |
|---|---|
| `--tmux` without `--worktree` | `Error: --tmux requires --worktree` |
| Untrusted directory | `Workspace trust not yet accepted. Run claude once in this directory and accept the trust dialog, then retry with --worktree.` |
| No git repo | `Error: --worktree requires a git repository` |
| No tmux | `Error: tmux is not installed. Install tmux with: brew install tmux` (`sudo apt install tmux` on Linux) |
| Windows | `Error: --tmux is not supported on Windows` |

#### Exit behavior

📦 `WorktreeExitDialog.tsx` offers three choices: keep worktree + tmux (prints
`tmux attach -t <name>`), keep worktree and kill tmux, or remove both.
`ExitWorktreeTool/prompt.ts`: the tmux session is killed on `remove` and left running on
`keep`, with its name returned so the user can reattach.

### tmux detection internals

📦 `src/utils/swarm/backends/detection.ts` — worth knowing because it is deliberately
narrow:

- `ORIGINAL_USER_TMUX` and `ORIGINAL_TMUX_PANE` are captured **at module load**, because
  `Shell.ts` later overwrites `TMUX` when Claude's own socket initializes.
- `isInsideTmux()` checks **only** the `TMUX` env var. The comment is explicit that it
  does *not* fall back to `tmux display-message`, because that succeeds whenever *any*
  tmux server is running on the box, not just when this process is inside one.
- `isTmuxAvailable()` = `tmux -V` exits 0.
- `getLeaderPaneId()` returns `TMUX_PANE` (e.g. `%0`).

Backends live in `src/utils/swarm/backends/`: `TmuxBackend.ts`, `ITermBackend.ts`,
`InProcessBackend.ts`, `PaneBackendExecutor.ts`, `registry.ts`, `detection.ts`,
`it2Setup.ts`, `teammateModeSnapshot.ts`, `types.ts`.

### Transcript storage

📖 + 🧪 `~/.claude/projects/<project>/<session-id>.jsonl`, where `<project>` is the working
directory path with every non-alphanumeric character replaced by `-`.

**Verified empirically** — this shell transform reproduces the real directory name exactly:

```sh
pwd | sed 's/[^a-zA-Z0-9]/-/g'
# /Users/bytedance/code/github.com/anthropics
#   → -Users-bytedance-code-github-com-anthropics
```

So this lists a directory's sessions newest-first, which is how you recover a session ID
that the picker will not show you:

```sh
ls -t ~/.claude/projects/$(pwd | sed 's/[^a-zA-Z0-9]/-/g')/*.jsonl | head -5
```

📖 Names longer than 200 characters are truncated and suffixed with a hash of the full path.
Retention is `cleanupPeriodDays`, default 30. `CLAUDE_CONFIG_DIR` moves the root;
`CLAUDE_CODE_PROJECT_DIR_NAME` (v2.1.234+) names the project directory explicitly.

### The installer (`https://claude.ai/install.sh`)

🧪 217 lines. `https://claude.ai/install.sh` answers **302**, so follow redirects.

```sh
DOWNLOAD_BASE_URL="https://downloads.claude.ai/claude-code-releases"
DOWNLOAD_DIR="$HOME/.claude/downloads"
```

Flow:

1. Refuse to run under sudo when `id -u` is 0 **and** `SUDO_USER` is set and is not `root`.
   Override with `CLAUDE_INSTALL_ALLOW_SUDO=1`. Plain root (containers, CI) is unaffected.
2. Pick `curl` or `wget`; use `jq` if present, else parse the manifest in pure bash.
3. Fetch `<base>/<version>/manifest.json`, read `.platforms[<platform>].checksum`, and
   require exactly 64 hex characters.
4. Download `<base>/<version>/<platform>/claude`, verify SHA-256 (`shasum -a 256` on
   darwin, `sha256sum` on linux). Delete and abort on mismatch.
5. `chmod +x`, then run `"$binary_path" install [target]` — the **binary installs itself**;
   the script only fetches and verifies it. Then the download is deleted.
6. On failure with code ≥ 128 and a tty, run `stty sane` first, because a signal death mid-TUI
   leaves the terminal in raw mode.
7. Exit **137** on Linux gets a specific message: the kernel OOM killer. It states Claude Code
   needs roughly **512MB** free to install. macOS has no equivalent, so the message is
   Linux-only.

🧪 Result layout: `~/.local/share/claude/versions/<version>`, with `~/.local/bin/claude`
as a symlink to it. Everything is under `$HOME`; no sudo, no npm global prefix.

### `--bare` mode

🔬 From `--help`, worth quoting because it is a big behavioral switch:

> Minimal mode: skip hooks, LSP, plugin sync, attribution, auto-memory, background
> prefetches, keychain reads, and CLAUDE.md auto-discovery. Sets `CLAUDE_CODE_SIMPLE=1`.
> Anthropic auth is strictly `ANTHROPIC_API_KEY`.

📖 Bare mode also does **not** bind the cross-session messaging inbox socket, so a bare
session is invisible to `ListAgents` and cannot receive messages.

### Cross-session messaging

📖 Mostly documented rather than binary-derived, but the env vars matter:

- `CLAUDE_CODE_MESSAGING_SOCKET` — per-session unix socket path, also shown in `/status`
  as the `Peer address` row with a `uds:` prefix.
- `CLAUDE_CODE_MESSAGING_TOKEN` — per-session token. A child process posting to its own
  session's socket sends `{"type":"auth","token":"<token>"}` as the first line.
- Same-machine delivery never leaves the box. Cross-machine goes through Anthropic servers
  over the target's Remote Control connection.
- Settings: `crossSessionInbound` (`accept`/`hold`/`refuse`), `isolatePeerMachines`,
  `dialogExpiry`.

### Hooks — the schema people get wrong

📖 A bare array of command strings is **invalid**. Hooks are matcher groups:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "…", "timeout": 30 }
        ]
      }
    ]
  }
}
```

Handler types: `command`, `http`, `mcp_tool`, `prompt`, `agent`. The `http` type takes
`url`/`headers`/`allowedEnvVars` and avoids shelling out to `curl` for webhook pings.

Event names: `SessionStart`, `Setup`, `SessionEnd`, `UserPromptSubmit`,
`UserPromptExpansion`, `Stop`, `StopFailure`, `PreToolUse`, `PostToolUse`,
`PostToolUseFailure`, `PermissionRequest`, `PermissionDenied`, `PostToolBatch`,
`Notification`, `MessageDisplay`, `SubagentStart`, `SubagentStop`, `TaskCreated`,
`TaskCompleted`, `TeammateIdle`, `InstructionsLoaded`, `ConfigChange`, `CwdChanged`,
`DirectoryAdded`, `FileChanged`, `WorktreeCreate`, `WorktreeRemove`, `PreCompact`,
`PostCompact`, `Elicitation`, `ElicitationResult`.

### Other things the binary gives up 🔬

Read out of `main.tsx`'s shipped action handler while grepping for something else. All
confirmed present in 2.1.237.

**Mode switches set env vars for their own children:**

```js
if (t.bare) process.env.CLAUDE_CODE_SIMPLE = "1"
if (yd())  process.env.CLAUDE_CODE_SAFE_MODE = "1",
           process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS = "1"   // + telemetry "startup_safe_mode"
```

**Third-party provider gates** — each has a matching skip-auth escape hatch:

| Use | Skip auth |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK` | `CLAUDE_CODE_SKIP_BEDROCK_AUTH` |
| `CLAUDE_CODE_USE_ANTHROPIC_AWS` | `CLAUDE_CODE_SKIP_ANTHROPIC_AWS_AUTH` |
| `CLAUDE_CODE_USE_MANTLE` | `CLAUDE_CODE_SKIP_MANTLE_AUTH` |
| `CLAUDE_CODE_USE_VERTEX` | `CLAUDE_CODE_SKIP_VERTEX_AUTH` |
| `CLAUDE_CODE_USE_ANTHROPIC_GOOGLE_CLOUD` | `CLAUDE_CODE_SKIP_ANTHROPIC_GOOGLE_CLOUD_AUTH` |

`CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` short-circuits startup prefetch — handy for
benchmarking cold start.

**Deep-link flags** (undocumented): `--prefill-b64`, `--deep-link-cwd-b64`, and a
`deepLinkOrigin` marker. Both b64 values are validated and ignored with a logged error
when malformed, so a bad link degrades rather than crashes.

**Easter egg** 🧪: running `claude code` treats `code` as a prompt, emits
`tengu_code_prompt_ignored`, and prints
`Tip: You can launch Claude Code with just \`claude\``.

**Telemetry event names** seen near this code, useful for grepping behavior:
`tengu_cli_flags`, `tengu_single_word_prompt`, `tengu_code_prompt_ignored`,
`tengu_worktree_removed`, `tengu_worktree_stale_lock_released`, `tengu_drift_lantern`.

**Agent-worktree lifecycle** functions worth knowing by name, since the docs describe the
behavior but not the machinery: `removeAgentWorktree`, `cleanupStaleAgentWorktrees`,
`releaseStaleClaudeWorktreeLocks`, `reapJobWorktreeIfSafe`. Agent worktrees are named
`agent-<id>`. Removal refuses when changed files would be lost, verifies the realpath still
matches, and restores `extensions.worktreeConfig` on the parent repo once its last linked
worktree is gone.

### Useful `strings` recipes

The binary is ~317MB, so grep it rather than reading it:

```sh
bin=$(readlink ~/.local/bin/claude)

# confirm a flag or subcommand really exists in this build
strings "$bin" | rg -- '--tmux=classic|--sdk-url'

# find the exact wording of an error before quoting it
# anchor the pattern -- 'is not installed' alone hits unrelated Git Bash and
# bunfig strings
strings "$bin" | rg 'Error: --tmux|Error: tmux is not installed'

# enumerate env vars -- 2.1.237 exposes 447 distinct CLAUDE_CODE_* names
strings "$bin" | rg '^CLAUDE_CODE_[A-Z_]+$' | sort --unique

# debug-log prefixes reveal subsystem names
strings "$bin" | rg '^\[bridge:'
```

Pair this with `rg` over `src/` for the mechanism, then confirm the user-visible strings
against the binary. `src/` explains *why*; the binary proves *what ships*.

### Auto-update bookkeeping 🧪

`~/.claude/.last-update-result.json` records the native updater's last outcome:

```json
{"timestamp":"2026-08-20T03:40:08.255Z","path":"native","outcome":"success","status":"success",
 "version_from":"2.1.236","version_to":"2.1.237","error_code":null}
```

Superseded versions are retained in `~/.local/share/claude/versions/` (2.1.234 … 2.1.237 all
present, ~310–317MB each), with `~/.local/bin/claude` symlinked to the active one. ⇒ A rollback
is just repointing that symlink, and the directory grows ~300MB per version kept.

### Workspace trust gates Remote Control 🧪🔬

🧪 `claude remote-control` refuses to start in a directory Claude has never run in:

```
Error: Workspace not trusted. Please run `claude` in <dir> first to review and accept
the workspace trust dialog.
```

🔬 The binary holds a **second, harsher variant** for `$HOME`:

> `Error: Workspace not trusted. <dir> is your home directory, and for security
> home-directory trust is never saved, so running `claude` here first won't help.
> Run `claude rc` from a project directory instead (run `claude` there once to accept
> the trust dialog).`

⇒ Two consequences. Home-directory trust is **never persisted**, by design. And there is
**no CLI flag** to pre-trust a directory — the only non-interactive route is the config
file. 🔬 Adjacent config keys in the same string table: `setTrustAccepted`, `projects`,
`hasTrustDialogAccepted`.

🧪 Writing this into `~/.claude.json` makes `remote-control` start without the dialog:

```json
{ "projects": { "/abs/path/to/repo": { "hasTrustDialogAccepted": true } } }
```

The path must be the **resolved** path. On a box where `$HOME` is a symlink
(`/home/user` → `/data00/home/user`), the error message prints the resolved form; that is
the key the config wants.

🔬 `claude --help` on `-p/--print`: *"The workspace trust dialog is skipped when Claude is
run in non-interactive mode."* ⇒ trust is an interactive-only gate, so `--print` sessions
never hit it and never record acceptance either.

### `claude rc` is a real subcommand 🔬

🧪 `claude rc --help` prints the full Remote Control help (`Remote Control - Control local
sessions from claude.ai/code or the Claude mobile app`). It is **not** in the `Commands:`
list of `claude --help`, alongside the already-noted hidden `remote-control`.

⚠️ Do not confuse it with the `--rc` *flag*, documented above as an alias for
`--remote-control`. `claude rc` is the **server**; `claude --rc` is an interactive session
that is also remotely controllable. The binary's own home-directory trust error recommends
`claude rc`, which is the strongest evidence it is a supported spelling rather than a leftover.

### Server startup: consent, banner, runtime keys 🧪🔬

🧪 First run in a directory prompts once:

```
Enable Remote Control? (y/n)
```

🔬 The nearby config key is `remoteDialogSeen`, so the consent is remembered, not re-asked.
🔬 Other strings in the same block: `Choose [1/2] (default: 1): `, and the analytics events
`tengu_bridge_started` / `tengu_bridge_spawn_mode_toggled`.

🧪 On success it prints a banner, and the environment URL is the shareable handle:

```
·✔︎· Connected · dotfiles
    Capacity: 1/32 · New sessions will be created in an isolated worktree
    devbox-dotfiles
Continue coding in the Claude mobile app or
https://claude.ai/code?environment=env_0146ofaCdJYkt9Pdc3SkUb66
space to show QR code · w to toggle spawn mode
```

🔬 The full runtime-key string set is `space to show QR code` / `space to hide QR code` /
` w to toggle spawn mode`, plus the status words `Reconnecting`, `retrying in `,
`disconnected `, `Attached`, and a `single-session` variant reading
`Single session <name> exits when complete`. ⇒ `--spawn=session` renders a different banner.

### `sshConfigs` — the Desktop/CLI SSH environment feature 🔬

🔬 The settings schema is in the binary, un-minified below. `O()` is a string schema, `zt()`
boolean, `Qe()` number, `$r([…])` enum —
read off neighbouring settings in the same table. `mt(ye({…}))` is an array-of-object, inferred
from the plural key name and the "configurations" wording.

```ts
sshConfigs: z.array(
  z.object({
    id: z.string()
      .describe('Unique identifier for this SSH config. Used to match configs across settings sources.'),
    name: z.string()
      .describe('Display name for the SSH connection'),
    sshHost: z.string()
      .describe('SSH host in format "user@hostname" or "hostname", or a host alias from ~/.ssh/config'),
    sshPort: z.number().int().optional()
      .describe('SSH port (default: 22)'),
    sshIdentityFile: z.string().optional()
      .describe('Path to SSH identity file (private key)'),
    startDirectory: z.string().optional()
      .describe(
        'Default working directory on the remote host. Supports tilde expansion (e.g. ~/projects). ' +
        'If not specified, defaults to the remote user home directory. Can be overridden by the ' +
        '[dir] positional argument in `claude ssh <config> [dir]`.',
      ),
  }),
).optional()
  .describe(
    'SSH connection configurations for remote environments. Typically set in managed settings ' +
    'by enterprise administrators to pre-configure SSH connections for team members.',
  )
```

Three things fall out of that:

1. The key is **`sshIdentityFile`**, not `identityFile`. Easy to get wrong.
2. A `claude ssh <config> [dir]` command exists. 🧪 It is **not registered on this
   `linux-x64` build** — `claude --help` lists no `ssh` under `Commands:`. ⇒ It is a
   *client-side* command (Desktop / macOS build); the Linux box is only ever the target.
3. Despite the "enterprise administrators" wording, the schema lives in the ordinary user
   settings table, so a personal user can set it in `~/.claude/settings.json`.

🔬 How the remote session gets credentials, from a warning string:

> ` ANTHROPIC_UNIX_SOCKET is set (claude ssh remote), and the local proxy is API-key-authed.`
> ` Unset ANTHROPIC_API_KEY / apiKeyHelper / ANTHROPIC_AUTH_TOKEN `

⇒ `claude ssh` forwards a **unix-domain socket** to the remote and points the remote's API
client at it, so the remote does not need its own login. The warning fires when that proxy is
API-key-authed while the remote also has its own key set.

🔬 One sibling string, worth knowing before you plan a deployment:

> ` This session is connected through an enterprise cloud gateway (set up via /login), which
> does not support Remote Control.`

⇒ Gateway auth and Remote Control are mutually exclusive.

---

## Claude Desktop 1.32885.1 (macOS)

Not the `claude` CLI, but the same family and the same disassembly method. Bundle:
`/Applications/Claude.app`, ~835MB. The interesting code is one Electron archive:
`Contents/Resources/app.asar`. It contains `\0` bytes, so `rg` needs `--text`.

### Desktop SSH does not support Kerberos / GSSAPI 🔬

Desktop's "Add SSH connection" does **not** shell out to `/usr/bin/ssh`. It bundles the
`ssh2` npm library and speaks the SSH protocol itself, in a class it logs as
`[SSH2Connection]`.

`ssh2` implements `none`, `publickey`, `password`, `keyboard-interactive`, `hostbased`
and agent auth. It has **no GSSAPI support**, and neither does the bundle:

```sh
rg --text --only-matching --no-filename \
   --regexp 'gssapi-with-mic' --regexp 'GSSAPIAuthentication' \
   /Applications/Claude.app/Contents/Resources/app.asar | sort | uniq --count
# (no output — zero matches)
```

⇒ Against a `GSSAPIAuthentication yes` + `PubkeyAuthentication no` +
`PasswordAuthentication no` server, Desktop can never authenticate. It exhausts its
methods, then falls back to prompting for a password the server will also refuse.
🧪 Observed against a ByteDance devbox; the system `ssh` succeeded from the same host
in the same minute over `gssapi-with-mic`.

### There is a second, OpenSSH-binary transport — behind an account flag 🔬

An `OpenSSHConnection` class exists that drives the real `ssh` binary with
`ControlMaster`, `-S <socket>`, `BatchMode=yes`, `ForwardAgent=no`, and (for the master
connection) `GSSAPIDelegateCredentials=no`. That path **would** support Kerberos.

Which transport runs is decided at connect time. 🔬 Un-minified from `app.asar`
(chunk `index2.chunk-zHVVshID.js`):

```js
const SSH_TRANSPORT_ENV_VAR = 'CLAUDE_DESKTOP_SSH_TRANSPORT'

// The env override is honored ONLY on internal ("Nest") or dev builds.
function sshTransportOverrideFromEnv() {                        // was: n
  if (!isInternalBuild() && !isDevBuild()) return undefined
  const value = process.env[SSH_TRANSPORT_ENV_VAR]
  return value === 'openssh' || value === 'ssh2' ? value : undefined
}

function selectSshTransport() {                                 // was: p
  if (isFeatureEnabled('291584251')) return 'ssh2'   // kill switch wins over everything
  const override = sshTransportOverrideFromEnv()
  if (override !== undefined) return override
  return isFeatureEnabled('3946462706') ? 'openssh' : 'ssh2'    // rollout flag
}

// Only wait on the flag service when no env override settles it locally.
function transportDependsOnAccountFlags() {                     // was: u
  return sshTransportOverrideFromEnv() === undefined
}
```

<details><summary>raw minified</summary>

```js
var t=`CLAUDE_DESKTOP_SSH_TRANSPORT`;function n(){if(!e.nC()&&!e.eC())return;let n=process.env[t];return n===`openssh`||n===`ssh2`?n:void 0}
function p(){if(e.By(`291584251`))return`ssh2`;let t=n();return t===void 0?e.By(`3946462706`)?`openssh`:`ssh2`:t}
function u(){return n()===void 0}
```
</details>

⇒ On a public release build, `CLAUDE_DESKTOP_SSH_TRANSPORT=openssh` is a **no-op**.
The gate waits up to 5s (`a=5e3`) for flags to settle, showing "Loading settings…".
Telemetry `desktop_ssh_connected` reports `ssh_transport` and `ssh_transport_flag_state`
(`fresh` | `cached` | `none`).

### `ProxyCommand` is supported, `ProxyJump` is not 🔬

The `ssh2` path parses `~/.ssh/config` (it logs how many `identityFiles` it resolved) and
honors `ProxyCommand`, spawning it as `sh -c <command>` and using its stdio as the socket.
It expands `%h`, `%p`, `%r`, `%n`. `ProxyJump` throws:

> `SSH host <host> uses ProxyJump (<value>), which is not yet supported. Consider using ProxyCommand instead.`

⚠️ `sshHostAllowlist` validates only the **resolved** `HostName`. The bundle itself warns
that a `ProxyCommand`/`ProxyJump` can therefore reach elsewhere — its own comment points at
`assertResolvedSshTargetAllowed` for the threat model.

### Desktop logs every SSH attempt 🔬

`~/Library/Logs/Claude/ssh.log` — full `ssh2` packet trace (`Outbound: Sending
USERAUTH_REQUEST (...)`, `Inbound: Received USERAUTH_FAILURE (...)`), plus
`[RemoteServerController]` connect/reconnect/flap decisions and `[BinaryDeployment]`.
Sibling logs: `main.log`, `mcp.log`, `coworkd.log`, `cowork_vm_{node,swift}.log`.

Reconnect behavior seen there: auth failures abort auto-reconnect after **1** attempt
unless live processes exist; repeated drops trigger flap detection with exponential
backoff (`FLAP_BACKOFF_BASE_MS`, `FLAP_BACKOFF_CAP_MS`, `FLAP_WINDOW_MS`).

### Other Desktop env vars found 🔬

`CLAUDE_DESKTOP_IS_NEST_BUILD`, `CLAUDE_DESKTOP_BACKGROUND_LAUNCH`,
`CLAUDE_DESKTOP_GITHUB_MCP_BINARY`, `CLAUDE_DESKTOP_LOCAL_FRAME_SHELL`,
`CLAUDE_DESKTOP_RESOLVING_ENVIRONMENT`.

### Recipe

```sh
ASAR=/Applications/Claude.app/Contents/Resources/app.asar
rg --text --only-matching --no-filename '.{300}<symbol>.{600}' "$ASAR" | head
```

---

## 2.1.246

Recorded **2026-08-25** on the veLinux devbox.

```
Running: native (2.1.246)
Commit:  1ba9d2211ae1
Platform: linux-x64
Path:    ~/.local/share/claude/versions/2.1.246
```

### Claude in Chrome: what actually turns it on

The question that produced this section: can a `claude remote-control` server give its
spawned sessions the browser tools? There is no flag for it, so the answer had to come
out of the binary.

🧪 The subcommand rejects the flag outright:

```
$ claude remote-control --chrome
Error: Unknown argument: --chrome
```

🔬 `--chrome` and `--no-chrome` are registered on the **top-level** command only:

```js
.option("--chrome","Enable Claude in Chrome integration")
.option("--no-chrome","Disable Claude in Chrome integration")
```

🧪 So `claude --chrome --remote-control [name]` parses — but that is the single-session
form. The multi-session server (`claude remote-control` / `claude rc`) has no spelling for
it. 📖 anthropics/claude-code#74671 asks for exactly this and is open.

#### The enable predicate 🔬

*Un-minified.* The helpers `E()`,
`d()`, and `_()` are the chunk's own; `E()` is an OAuth-scope check and `d()` reads the
global config. **`_()` I did not resolve** — it is a negative guard that sits between the
env var and the config key.

```js
function isClaudeInChromeEnabled(flagValue) {          // was: Je(e)
  // 1. OAuth scope. The message names the accepted scopes, which is the useful part.
  if (!hasAcceptedOAuthScope()) {
    log("[Claude in Chrome] Disabled: OAuth token has no scope accepted by " +
        "/api/oauth/validate (needs user:profile, user:office, or user:ccr_inference; " +
        "env-var and setup-token sessions default to user:inference only)")
    return false
  }
  // 2. The CLI flag, both ways.
  if (flagValue === true)  return true                 // --chrome
  if (flagValue === false) return false                // --no-chrome
  // 3. The environment variable, both ways. CFC = Claude For Chrome.
  if (env.CLAUDE_CODE_ENABLE_CFC === true)  return true
  if (env.CLAUDE_CODE_ENABLE_CFC === false) return false
  // 4. An unresolved guard.
  if (unresolvedGuard()) return false
  // 5. The /chrome menu's "Enabled by default" toggle, from the global config.
  const config = getGlobalConfig()
  if (config.claudeInChromeDefaultEnabled !== undefined)
    return config.claudeInChromeDefaultEnabled
  return false
}
```

<details><summary>Original minified form, as it appears in the binary</summary>

```js
function Je(e){if(!E())return i("[Claude in Chrome] Disabled: OAuth token has no scope accepted by /api/oauth/validate (needs user:profile, user:office, or user:ccr_inference; env-var and setup-token sessions default to user:inference only)"),!1;if(e===!0)return!0;if(e===!1)return!1;if(m.CLAUDE_CODE_ENABLE_CFC===!0)return!0;if(m.CLAUDE_CODE_ENABLE_CFC===!1)return!1;if(_())return!1;let o=d();if(o.claudeInChromeDefaultEnabled!==void 0)return o.claudeInChromeDefaultEnabled;return!1}
```

</details>

⇒ **`CLAUDE_CODE_ENABLE_CFC=1` is the flag's replacement for any command that will not take
`--chrome`.** A `remote-control` server started with it in its environment passes it to every
child session it spawns, since 📦 `sessionRunner.ts` spawns plain child `claude` processes.

⚠️ Not verified end-to-end here: the devbox has no Chrome extension installed, so no session
on it has actually wired the MCP server. 🧪 What was verified is that the variable reaches the
server process (`/proc/<pid>/environ` on three restarted `claude remote-control` servers).

⚠️ Precedence detail worth keeping: the env var is read **before** the config key, and both
`true` and `false` are honoured at each step. `CLAUDE_CODE_ENABLE_CFC=false` therefore beats
the `/chrome` toggle, and `--no-chrome` beats the env var.

#### Wiring, and what suppresses it 🔬

*Un-minified* from the startup path. `parsedOptions` is Commander's option object.

```js
const chromeEnabled =
  isClaudeInChromeEnabled(parsedOptions.chrome) && secondGate()

const deniedByPolicy   = isMcpServerDenied(serverName, serverConfig())
const enterpriseBlocks = hasEnterpriseMcpConfig() || deniedByPolicy

// Note both skips require the opt-in to have been IMPLICIT: an explicit --chrome or
// CLAUDE_CODE_ENABLE_CFC=1 does not take these branches.
const skipForPolicy =
  chromeEnabled && parsedOptions.chrome !== true &&
  env.CLAUDE_CODE_ENABLE_CFC !== true && enterpriseBlocks

const skipForSafeMode =
  chromeEnabled && parsedOptions.chrome !== true &&
  env.CLAUDE_CODE_ENABLE_CFC !== true && isSafeMode()
```

🔬 The three messages that come out of it, verbatim:

```
[Claude in Chrome] Skipping chrome wiring: blocked by enterprise MCP config or managed deniedMcpServers policy
[Claude in Chrome] Skipping chrome wiring: --safe-mode disables MCP
MCP server blocked by enterprise policy: <name>
```

⇒ Ask explicitly and a policy block becomes a **warning** rather than a silent skip: the
`skipFor*` branches are bypassed and the flow falls through to the third message.

`secondGate()` (`mo()`) is an additional condition I did not identify; three different
functions named `mo` exist in the bundle, so the name alone does not resolve it.

#### The MCP server is the binary talking to itself 🔬

```js
function getClaudeInChromeMcpServerConfig() {           // was: xe()
  return {
    type: "stdio",
    command: process.execPath,                          // the claude binary
    args: ["--claude-in-chrome-mcp"],
    scope: "dynamic",
  }
}
```

🔬 `process.argv[2] === "--claude-in-chrome-mcp"` is dispatched at the very top of the
entrypoint, before the CLI is built, to `runClaudeInChromeMcpServer` (telemetry:
`cli_claude_in_chrome_mcp_path`). A sibling argv mode `--chrome-native-host` is what the
installed native-messaging manifest points at.

🔬 Native-messaging install, done as a side effect of wiring:

```json
{
  "name": "com.anthropic.claude_code_browser_extension",
  "description": "Claude Code Browser Extension Native Host",
  "path": "<claude binary> --chrome-native-host",
  "type": "stdio",
  "allowed_origins": ["chrome-extension://fcoeoabgfenejglbffodgkkbkcdhcgfn/"]
}
```

- File name is `<name>.json`; on Windows the directory is
  `%APPDATA%` (or `<home>/AppData/Local`) + `Claude Code/ChromeNativeHost`, otherwise a
  per-browser list.
- First-time install opens `https://clau.de/chrome/reconnect` in the browser.
- 🔬 When a session runs with permissions bypassed, the MCP server is handed
  `CLAUDE_CHROME_PERMISSION_MODE=skip_all_permission_checks` in its env — the browser
  tools have their own permission mode, separate from the session's.

#### `isRemoteMode` is about **cloud** sessions, not Remote Control 🔬

Two unrelated "remote" concepts share the word, and only one of them touches Chrome.

```js
isRemoteMode: env.CLAUDE_CODE_REMOTE || isCloudSession()
```

*Un-minified*, the predicate that consumes it:

```js
function shouldSuppressChromeOffer({                    // was: to({…}), called as Sp({…})
  isSSHPending, isRemoteMode, hasTeleport, isSafeMode,
  permissionMode, isBypassPermissionsModeAvailable, teammateAgentId,
}) {
  return isSSHPending || isRemoteMode || hasTeleport || isSafeMode ||
    permissionMode === "bypassPermissions" ||
    (permissionMode === "plan" && isBypassPermissionsModeAvailable) ||
    teammateAgentId !== undefined
}
```

🔬 Its only use at the call site is `suppress ? false : offerClaudeInChrome` — it gates the
**onboarding offer**, never the tools. ⇒ A `claude remote-control` session is an ordinary
local session for Chrome purposes: the browser it drives is the one on the machine running
the server, not one on the phone that is driving the session. A **cloud** session
(`CLAUDE_CODE_REMOTE`) is the case with no local browser.

#### Config keys and tool prefixes 🔬

Global config (`~/.claude.json`) keys in this area:

```
claudeInChromeDefaultEnabled        the /chrome "Enabled by default" toggle
hasCompletedClaudeInChromeOnboarding
cachedChromeExtensionInstalled      lets startup skip a probe
chromeExtension.pairedDeviceId      a browser paired on ANOTHER device
```

🔬 The permission layer knows four tool-name prefixes for the same feature, plus a
`remote-devices` carrier:

```js
["mcp__claude-in-chrome__", "mcp__Claude_in_Chrome__",
 "mcp__remote-devices__claude-in-chrome__", "mcp__remote-devices__Claude_in_Chrome__",
 "mcp__Claude_Preview__", "mcp__Claude_Browser__", "mcp__remote-devices__Claude_Browser__"]
```

⇒ Together with `chromeExtension.pairedDeviceId`, that is the seam for driving a browser
attached to a *different* machine than the one running Claude. Not exercised here.

🔬 The `/chrome` menu's own entries: `install-extension`, `select-browser`, `reconnect`,
`toggle-default` (rendered as `Enabled by default: Yes|No`).

### Transcript token accounting — what the JSONL really records 🧪🔬📦

Findings from writing `transcript-usage.ts`, a report that re-derives `/context` and the
session cost from the raw transcript. Source and tests live in
`~/code/github.com/vegerot/coding-model-router/tools/transcript-usage.{ts,test.ts}`;
the `context-deep` skill (`~/.claude/skills/context-deep/SKILL.md`) is the wrapper.
All 🧪 claims below were re-verified on 2.1.246 against a live session transcript.

#### The shape of a conversation on disk

🧪 One session is **one JSONL file** — `~/.claude/projects/<sanitized-cwd>/<session-id>.jsonl`,
append-only, one JSON object per line, no wrapping array and no trailing comma. See
[Transcript storage](#transcript-storage) under 2.1.237 for the path transform and retention.

⇒ Append-only means the file is a **log, not a document**. Rewound branches, abandoned tool
calls, and superseded state all stay in it. The live conversation is a *subset* you have to
reconstruct (see `parentUuid` below).

🧪 Rows fall into two families. Only the first is the conversation:

| Family | `type` values | Has `uuid` / `parentUuid` / `timestamp` |
|---|---|---|
| **Conversation entries** | `user`, `assistant`, `attachment`, `system` | yes |
| **Session state** | `mode`, `last-prompt`, `ai-title`, `atis-latch`, `bridge-session`, `file-history-snapshot` | no |

🧪 Session-state rows are tiny bookmarks re-appended whenever the value changes, keyed only by
`sessionId`. A session that switched mode 11 times has 11 `mode` rows; the last one wins.

```jsonc
{"type":"mode","mode":"normal","sessionId":"d3554ff7-…"}          // permission mode
{"type":"last-prompt","lastPrompt":"…","leafUuid":"784f67ee-…"}   // resume bookmark
{"type":"ai-title","aiTitle":"Consolidate transcript usage…"}      // the picker's label
{"type":"bridge-session","bridgeSessionId":"…","lastSequenceNum":…}// Remote Control cursor
{"type":"file-history-snapshot","messageId":"…","snapshot":{…}}    // rewind/undo checkpoint
{"type":"atis-latch","atis":"v1.…"}                                // opaque token
```

⇒ `leafUuid` on `last-prompt` is the anchor `--continue` resumes from. `ai-title` is why the
session picker shows prose instead of a UUID.

##### The envelope every conversation entry carries

🧪 On 2.1.246 these keys appear on `user`, `assistant`, and `attachment` rows alike:

| Field | Why it matters |
|---|---|
| `uuid` | This entry's id |
| `parentUuid` | The entry before it. `null` only at the root |
| `type` | `user` / `assistant` / `attachment` / `system` |
| `timestamp` | ISO 8601, UTC |
| `sessionId` | Also duplicated as `session_id` on some rows |
| `version` | The Claude Code build that wrote the line — one file can span upgrades |
| `cwd`, `gitBranch` | Where the turn ran |
| `isSidechain` | `true` ⇒ a subagent wrote it, not the main loop |
| `userType` | `external` for a real person |
| `entrypoint` | How the session started (`cli`, …) |

⇒ `parentUuid` makes the file a **tree, not a list**. Walking it backwards from the last
`assistant` entry yields exactly the conversation that was last sent; any entry not on that
chain is a branch that was rewound and is still costing disk but not context.

##### Per-type payload

🧪 **`assistant`** — adds `message` (`{id, model, content[], usage}`), `effort`, and
`requestId`. `message.content` is the block array: `text`, `thinking`, `tool_use`.

🧪 **`user`** — `message.content` is either a plain **string** (a typed prompt) or a block
array holding **`tool_result`** blocks. A tool result is a *user* row, because that is how the
API models it. Extra fields worth knowing:

| Field | Meaning |
|---|---|
| `toolUseResult` | Sibling of `message`. The **full local** result record |
| `sourceToolAssistantUUID` | The `assistant` entry whose `tool_use` this answers |
| `promptId`, `promptSource` | `promptSource: "typed"` for a real keystroke prompt |
| `origin` | `{"kind":"human"}` — separates a person from a replayed/automated prompt |
| `permissionMode` | The mode in force when the prompt was sent, e.g. `auto` |
| `isMeta` | System-injected pseudo-prompt, not something the user typed |
| `isCompactSummary` | This row is a compaction summary standing in for older turns |

⚠️ 🧪 **`toolUseResult` is not what the model saw.** It is the untruncated local record;
`message.content` holds the possibly-truncated version actually sent. Measured in one session:
138,028 bytes of `toolUseResult` against 108,754 bytes of matching `content` (1.27x), and a
single `Bash` row where a large output was spilled to a file stored **32,061 bytes locally
against 2,457 sent** — 13x. A parser that reads `toolUseResult` for token accounting will
overcount, sometimes wildly.

🧪 **`attachment`** — `attachment: {type, …}`, no `message` at all. This is context Claude Code
injects around the conversation: `skill_listing`, `deferred_tools_delta`, `async_hook_response`,
`total_tokens_reminder`, and more. Field names differ per subtype.

🧪 **`system`** — local notices, keyed by `subtype` (`local_command`, `stop_hook_summary`,
`turn_duration`), usually `isMeta: true`. Carries `level` (`info` / `suggestion`) and subtype
specific fields such as `durationMs`, `messageCount`, `hookErrors`, `toolUseID`.

##### What actually fills the file

🧪 One 498KB session, by row type:

```
type                     rows      bytes
user                       26    260,235   ← tool results + toolUseResult duplication
attachment                117    128,051
assistant                  38     86,133
last-prompt                10      3,243
bridge-session             11      2,926
atis-latch                 10      2,710
system                      3      2,452
ai-title                   10      1,320
mode                       11        902
file-history-snapshot       3        705
```

⇒ `user` rows dominate, and they are almost entirely tool output, not typing. `assistant` rows
are a sixth of the file despite being the only ones that cost output tokens.

⇒ **File size is a bad proxy for context cost**, in both directions: `toolUseResult` inflates
it, while the system prompt, tool definitions, memory, and skills cost tokens and appear
nowhere.

#### One API response writes many JSONL lines — dedupe by `message.id`

🧪 Claude Code writes **one line per content block**, and every one of those lines repeats
the *same* `message.usage` object. A thinking + text + tool_use turn is 3 lines, all carrying
the full token count of the request.

```sh
jq -r 'select(.type=="assistant") | .message.id' "$T" | sort | uniq -c | sort -rn | head
#       3 msg_011CeQ8m8x1nasj2RkdABA9C
#       3 msg_011CeQ8kUeuEQCjmsFAYA8X2
```

⇒ Summing `usage` per line **overcounts by 2-3x**. Dedupe on `message.id` first. In the
session that produced this note, 26 assistant lines were 9 real API messages.

⇒ `model: "<synthetic>"` marks a local notice (an error, a cancel, an interrupt) that no API
call produced. Its `usage` is all zeros, but it still occupies an `assistant` line. Skip it.

#### The `usage` block carries more than the four public counters

🧪 A real 2.1.246 `message.usage`, reformatted:

```jsonc
{
  "input_tokens": 2,
  "cache_creation_input_tokens": 1557,
  "cache_read_input_tokens": 86802,
  "output_tokens": 612,
  "output_tokens_details": { "thinking_tokens": 117 },
  "server_tool_use": { "web_search_requests": 0, "web_fetch_requests": 0 },
  "service_tier": "standard",
  // The TTL split. A 1h write costs 2x input, a 5m write 1.25x, so the total alone
  // is not enough to price a message.
  "cache_creation": { "ephemeral_1h_input_tokens": 1557, "ephemeral_5m_input_tokens": 0 },
  "inference_geo": "not_available",
  // Per-request breakdown when one assistant turn made several API round trips.
  "iterations": [ { "input_tokens": 2, "output_tokens": 612, "type": "message", /* ... */ } ],
  // "fast" here is /fast mode, which bills at a premium tier. Recorded per message.
  "speed": "standard"
}
```

⇒ `thinking_tokens` is a **subset** of `output_tokens`, not an addition. Answer tokens are
`output_tokens - thinking_tokens`.

⇒ Older transcripts have `cache_creation_input_tokens` but no `cache_creation` split. Charge
those at the 5m rate.

#### Thinking is billed but stored empty

🧪 On Opus 5 every stored `thinking` block has `thinking: ""` and a non-empty `signature`:

```sh
jq -r '.message.content[]? | select(.type=="thinking")
       | "thinking_len=\(.thinking|length) sig_len=\(.signature|length)"' "$T"
# thinking_len=0 sig_len=488
# thinking_len=0 sig_len=2440
```

⇒ Any tool that measures context by reading block text reports **0 thinking tokens** while
the API billed thousands. The signature is kept so the block can be replayed; the reasoning
text is not. Do not read a `0` here as "thinking was free".

#### Where the per-message context size comes from

🧪 `effort` (`low | medium | high | xhigh | max`) sits on the `assistant` row, not inside
`message`. It is the only place the reasoning effort of a single message is recorded.

⇒ The context Claude Code held for one request is
`input_tokens + cache_read_input_tokens + cache_creation_input_tokens` of that assistant
message. There is no separate "context size" field.

#### Context estimation: 4 bytes per token, and a flat 2000 for images

📦 `src/services/tokenEstimation.ts:203` — un-minified, this is the whole estimator:

```ts
export function roughTokenCountEstimation(content: string, bytesPerToken: number = 4): number {
  return Math.round(content.length / bytesPerToken)
}
```

📦 `bytesPerTokenForFileType()` (same file) drops that to **2** for `json`, `jsonl`, and
`jsonc`, with the reason recorded: "Dense JSON has many single-character tokens (`{`, `}`,
`:`, `,`, `"`) which makes the real ratio closer to 2 rather than the default 4." The
comment says an underestimate "can let an oversized tool result slip into the conversation".

📦 `roughTokenCountEstimationForBlock()` (`tokenEstimation.ts:391`) returns a **flat 2000**
for `image` and `document` blocks, and the comment explains why the catch-all must not see
them: "a 1MB PDF is ~1.33M base64 chars → ~325k estimated tokens, vs the ~2000 the API
actually charges."

⇒ So `/context` image and PDF numbers are a constant, not a measurement. Real cost is
`⌈w/28⌉ × ⌈h/28⌉` patches after the documented downscale — measurable exactly from the PNG
IHDR / JPEG SOF / GIF / WebP header in the stored base64, which is what `transcript-usage.ts`
does.

⇒ `tool_use` is estimated as `name + jsonStringify(input)`, `tool_result` recurses into its
own `content`, and anything unrecognized falls through to `jsonStringify(block)`.

#### Hook stdout is local — only two fields reach the model

🧪 An `async_hook_response` attachment stores `stdout`, `stderr`, `exitCode`, `processId`,
`toolUseID`, `durationMs`. **None of that is sent.** Only
`response.systemMessage` and `response.hookSpecificOutput.additionalContext` are.

🧪 Measured: 52 `async_hook_response` attachments in one session, **0 tokens** of context.

⇒ A chatty hook is free as long as it stays silent in `response`. This contradicts the
intuition that hook output costs context.

#### Attachment types are where the invisible context lives

🧪 `type: "attachment"` entries, by subtype, from one 2.1.246 session:

```
attachment subtype          entries   tokens
skill_listing                     1    3,844
deferred_tools_delta              1    2,529
mcp_instructions_delta            1      784
agent_listing_delta               1      637
total_tokens_reminder             7      153
hook_success                      1       27
auto_mode                         1        6
async_hook_response              52        0
```

⇒ `skill_listing` and `deferred_tools_delta` together cost ~6.4k tokens at session start,
before any work happens. That is the price of installed skills plus the deferred-tool roster.

🧪 `total_tokens_reminder` is the mechanism behind the budget line the model sees:
`{"type":"total_tokens_reminder","text":"<total_tokens>14912564 tokens left</total_tokens>"}`.

#### Two thirds of a real context is not in the transcript

🧪 Live composition of the session that produced this note, at 91,545 tokens of context:

```
actual context:      91,545
estimated messages:  31,146  (34%)
unmeasured:          60,399  (66%)  = system prompt + tool defs + memory + skills + estimate error
```

⇒ The transcript stores the conversation, never the request preamble. System prompt, tool
definitions, `CLAUDE.md` memory, and skill bodies are reconstructed nowhere in the file, so
any transcript-derived report can only show them as a residual. That residual also absorbs
every error in the 4-bytes-per-token estimate, so do not quote it as exact.

⇒ Practical consequence: a session that "feels full" early is usually paying the fixed
preamble, not the messages. Trimming tool results helps the 34%, not the 66%.

#### Recipe

```sh
# The current session's transcript. CLAUDE_CODE_SESSION_ID is set in every session.
T="$(fd --type f "${CLAUDE_CODE_SESSION_ID}.jsonl" ~/.claude/projects)"

# Real API message count, not line count.
jq -r 'select(.type=="assistant" and .message.model != "<synthetic>") | .message.id' "$T" \
  | sort -u | wc -l

# Everything above, as a report.
transcript-usage "$T"          # add --json to compute on the numbers
```

⚠️ Claude Code transcripts only. Codex, Trae CLI, and opencode use a different JSONL shape
(`response_item` / `session_meta` events, no `type: "user"` / `"assistant"` with a `message`
field), so pointing a Claude-Code parser at one silently undercounts instead of erroring.

### Computer use: `--computer-use-mcp` is an entrypoint, not a switch 🔬📦🧪📖

Recorded **2026-08-26** on the work MacBook, same version (2.1.246).

The question: is there a config setting that permanently enables computer use, the way
`permissions.defaultMode` permanently enables bypass mode? Answering it from `src/` alone
produced a **wrong answer**, so the correction matters more than the finding.

**`--computer-use-mcp` does not enable anything.** 📦 `src/entrypoints/cli.tsx:86` matches it
positionally — `process.argv[2] === '--computer-use-mcp'` — and dispatches to
`runComputerUseMcpServer()`. It is the subprocess entrypoint for the MCP server itself, written
into the child args by `src/utils/computerUse/setup.ts:36`. Running `claude --computer-use-mcp`
by hand just starts a bare stdio MCP server. It is the same shape as the neighbouring
`--chrome-native-host` branch.

Amusingly, that spawn never happens: `setup.ts` says the `command`/`args` are *"never spawned —
client.ts intercepts by name and uses the in-process server. The config just needs to exist with
type 'stdio' to hit the right branch."* The MCP wrapper is not ceremony either — the comment
notes the API backend detects `mcp__computer-use__*` tool names and injects a CU availability
hint into the system prompt, which built-in tool names would not trigger.

**How it is actually enabled** 📖🧪 — `/mcp` → select `computer-use` → **Enable**. That persists
**per project**, in `~/.claude.json`, not `settings.json`:

```json
"projects": {
  "/path/to/project": { "enabledMcpServers": ["computer-use"] }
}
```

🧪 Confirmed by reading `~/.claude.json` directly: four projects carried the entry, and the
`mcp__computer-use__*` tools (24 of them) loaded into the running session. There is no global
key — `enabledMcpjsonServers` governs `.mcp.json` servers, not this built-in one.

#### ⚠️ Where `src/` is stale — and how the binary check went wrong

📦 `src/utils/computerUse/gates.ts` describes an **ant-only dogfooding gate**: GrowthBook
`tengu_malort_pedway` defaulting to `enabled: false`, a Max/Pro subscription check, and an
`ALLOW_ANT_COMPUTER_USE_MCP=1` escape hatch for ants whose shell inherited monorepo dev config
(detected via `MONOREPO_ROOT_DIR`).

📖 That is no longer the shipping state. Computer use is a **public research preview** — macOS
only in the CLI, Pro or Max, claude.ai auth (not Bedrock/Vertex/Foundry), interactive sessions
only, v2.1.85+. See <https://code.claude.com/docs/en/computer-use>.

🔬 The trap: `strings` on the installed 2.1.246 still finds both gate names.

```bash
BIN=~/.local/share/claude/versions/2.1.246
for s in computer-use-mcp tengu_malort_pedway ALLOW_ANT_COMPUTER_USE_MCP request_access; do
  printf '%-32s %s\n' "$s" "$(strings "$BIN" | grep --count --fixed-strings "$s")"
done
# computer-use-mcp                 3
# tengu_malort_pedway              2
# ALLOW_ANT_COMPUTER_USE_MCP       2
# request_access                   32
```

Those hits were read as *confirming* the snapshot's ant-only story. They do not. **A live string
proves the code exists, not that it is the path taken** — a shipped feature and the remains of
its rollout gate coexist happily in one binary. This is the counterpart to the
`switch-models-on-flag` failure recorded above: there the binary was never checked, here it was
checked and misread.

📖 `CHANGELOG.md` had the tell, and a too-narrow grep hid it — a single line, *"Fixed
`switch_display` in the computer-use tool returning 'not available in this session' on
multi-monitor setups"*. A **bugfix for a user-facing tool is evidence the tool ships.** When
`src/` says a feature is gated off, grep the changelog wide before repeating it.

### `permissions.defaultMode: "bypassPermissions"` replaces the CLI flag 📦

📦 `src/utils/permissions/permissionSetup.ts:722` builds `orderedModes` in this precedence:

```
--dangerously-skip-permissions  →  --permission-mode  →  settings permissions.defaultMode
```

`bypassPermissions` is pushed straight from settings, so the flag is **not** a prerequisite for
the settings path — despite the SDK-side error text in `cli/print.ts:4595` ("…because the
session was not launched with `--dangerously-skip-permissions`"), which guards only *runtime*
`setMode` requests. `bypassPermissions` is a member of `EXTERNAL_PERMISSION_MODES`
(`src/types/permissions.ts:16`), so it is valid in `settings.json`.

Two things suppress it:

- `permissions.disableBypassPermissionsMode: "disable"`, or the Statsig gate
  `tengu_disable_bypass_permissions_mode` — the gate outranks settings, and each produces a
  different notification string.
- `CLAUDE_CODE_REMOTE` — only `acceptEdits`, `plan`, and `default` survive there. The comment
  explains why: `bypassPermissions` "would otherwise silently grant full access in a remote
  environment". It logs `tengu_ccr_unsupported_default_mode_ignored` and falls through.

The companion key `skipDangerousModePermissionPrompt: true` suppresses the one-time acceptance
screen. 📦 `settings.ts:882` reads it from **user, local, flag, and policy** settings only —
project settings are deliberately excluded, the same anti-RCE reasoning as `hasAutoModeOptIn()`
directly below it.

🧪 Practical note on macOS: if `~/.claude/settings.json` is a dotfiles symlink, BSD `sed -i`
refuses it outright — *"in-place editing only works for regular files"*. Resolve with
`readlink -f` and edit the real file.

### Shell snapshots: how your rc file reaches the Bash tool 📦🧪

Recorded **2026-08-26** on the work MacBook, same version (2.1.246).

The Bash tool does **not** run `.zshrc` per command. Once per session Claude Code builds a
*shell snapshot* and every Bash call sources it.

📦 `src/utils/bash/ShellSnapshot.ts:456` — the snapshot shell is spawned as a login shell with
three variables forced in:

```js
execFile(binShell, ['-c', '-l', snapshotScript], {
  env: {
    ...(process.env.CLAUDE_CODE_DONT_INHERIT_ENV ? {} : subprocessEnv()),
    SHELL: binShell,
    GIT_EDITOR: 'true',
    CLAUDECODE: '1',
  },
  timeout: SNAPSHOT_CREATION_TIMEOUT,
  maxBuffer: 1024 * 1024,
})
```

**`CLAUDECODE=1` is set while the user's rc file is being read.** That makes it the supported
hook for "define this alias only in my real terminal" — e.g. keeping `alias rm='rm -i'` out of
Claude's non-TTY shell, where the `-i` prompt has nobody to answer it.

📦 `getConfigFile()` chooses the rc file by shell name only: `.zshrc` if the path contains `zsh`,
`.bashrc` if it contains `bash`, otherwise `.profile`. 📦 `getUserSnapshotContent()` then emits
three sections:

| Section | zsh | bash |
|---|---|---|
| `# Functions` | `typeset +f`, filtered by `grep -vE '^_[^_]'`, each dumped with `typeset -f` | `declare -F`, same filter, base64-encoded per function |
| `# Shell Options` | `setopt \| sed 's/^/setopt /'` | `shopt -p`, `set -o`, plus `shopt -s expand_aliases` |
| `# Aliases` | `alias \| sed 's/^alias //g' \| sed 's/^/alias -- /' \| head -n 1000` | same (with a `winpty` filter on msys/cygwin) |

⚠️ Note the **`head -n 1000` cap** on aliases, and the same cap on shell options. A very large rc
file can silently lose the tail.

🧪 On this machine the snapshot is `~/.claude/shell-snapshots/snapshot-zsh-<ms>-<id>.sh`, 4075
lines. Two traps when reading one:

- It **opens with `unalias -a 2>/dev/null || true`**, then re-adds everything. The snapshot is
  authoritative — nothing survives from outside it.
- Aliases are written as `alias -- rm='rm -i'`. Grepping a snapshot for `alias rm` returns
  nothing and looks like evidence the alias came from elsewhere.

🧪 The snapshot is built at session start, so an rc-file change does not affect the session that
is already running. Restart `claude`.

🧪 Reproducing the capture exactly, to test an rc-file guard without restarting:

```sh
CLAUDECODE=1 SHELL=/bin/zsh GIT_EDITOR=true /bin/zsh -c -l \
  'source ~/.zshrc >/dev/null 2>&1; alias | grep "^rm="'
```

⚠️ Test the **unset** branch too. A guard written `[ -z "$CLAUDECODE" ]` is fatal under `nounset`
(`CLAUDECODE: parameter not set`), which aborts the sourced file and silently discards every
alias defined below it. Use `${CLAUDECODE:-}`.

## 2.1.250

Recorded **2026-08-28** on the veLinux devbox, against
`~/.local/share/claude/versions/2.1.250` (224 MB). The active symlink had already moved to
2.1.251 by the time this was written up; the offsets below are for 2.1.250.

### The machine name at claude.ai/code is `os.hostname()`, full stop 🔬

The question: the environment list shows the box as `n251-236-182`. Can it say `devbox`?

🔬 `machineName` appears at 5 byte offsets and `machine_name` at 3. They are one path, with
nothing between the read and the wire:

```js
// 194865608 — the bridge chunk's import
import { hostname as or } from "os";

// 194935815 (`claude remote-control`) and 194947187 (the in-session bridge)
let lt = or();
let C = { dir: P, machineName: lt, machineId: Gt, branch: pt, gitRepoUrl: dt, ... };

// 194866944 — registration
rt.post(`${e.baseUrl}/v1/environments/bridge`, {
  machine_name: o.machineName,
  ...(o.machineId != null && { machine_id: o.machineId }),
  directory: o.dir, branch: o.branch, git_repo_url: xY(o.gitRepoUrl), ...
})
```

⇒ **No setting, no config key, no flag.** 🔬 `grep -E 'CLAUDE_[A-Z_]*MACHINE[A-Z_]*'` over the
whole binary returns nothing, so no env var overrides it either.

⚠️ `machine_id` is not the display name. It comes from `getOrCreateRemoteControlMachineId`
(`chunk-yvtrda2c.js`), awaited and persisted, and is independent of the hostname.

🎣 Two flags look like the answer and are not: `--name` sets the **session** title, and
`--remote-control-session-name-prefix` (env `CLAUDE_REMOTE_CONTROL_SESSION_NAME_PREFIX`) sets
the **session** name prefix. The prefix merely *defaults* to the hostname, which is what makes
it look right. Both are already in the 2.1.237 flag table above.

⇒ To rename the machine you must change what `gethostname(2)` returns:
`sudo hostnamectl set-hostname devbox` system-wide, or give the bridge its own UTS namespace.

### Faking the hostname for one process 🧪

🧪 The single-`unshare` form does **not** work:

```
$ unshare --user --uts --map-current-user bash -c 'hostname devbox'
CapPrm: 0000000000000000
CapEff: 0000000000000000
hostname: you must be root to change the host name
```

⇒ Mapping to your own UID keeps **no capabilities**, so `CAP_SYS_ADMIN` is unavailable and the
UTS namespace is useless. 🧪 `--map-auto` also fails on this box:
`unshare: failed to execute newuidmap: No such file or directory`.

🧪 Nesting works. Set the hostname as fake root, then map back to the real UID:

```bash
unshare --user --uts --map-root-user bash -c "
  hostname devbox
  exec unshare --user --map-user=$(id -u) --map-group=$(id -g) \
    claude remote-control
"
# → hostname=devbox uid=1001 owner=max.coplan
```

The UTS namespace survives the inner `unshare`; the fake root does not. Without the inner one
the process runs as UID 0 and `stat -c %U ~/.claude/settings.json` reports `root`.

### Reading a 224 MB binary without timing out ⚙️

🧪 `grep -a -o -E '.{600}machineName.{600}'` on this binary exceeds the 120 s Bash-tool timeout.
The working method is two steps:

```sh
timeout 110 grep --byte-offset --only-matching --text 'machineName' 2.1.250
dd if=2.1.250 bs=1 skip=$((OFFSET-500)) count=1000 2>/dev/null
```

⇒ `grep -abo` for the bare literal is fast; the context window comes from `dd`. Bounding the
offsets with `awk -F: '$1>194000000 && $1<195400000'` keeps the hits to one chunk — the bundle
is concatenated modules, so nearby offsets are the same module.

---

## 2.1.251

Recorded **2026-08-28** on the ByteDance work MacBook.

```
Running: native (2.1.251)
Commit:  37534ac596d8
Platform: darwin-arm64
Path:    ~/.local/share/claude/versions/2.1.251
```

The question that produced this section: *which flags and env vars make Claude Code more
powerful?* Everything below is un-minified.

### Settings load order

🔬 The source list is a literal array in the binary:

```js
["userSettings", "projectSettings", "localSettings", "flagSettings", "policySettings"]
```

`--settings <file-or-json>` is `flagSettings`. It sits **after** the three on-disk scopes, so
it merges over `~/.claude/settings.json` and overrides only the keys it names.
`policySettings` still outranks it. A `--settings` JSON string is therefore additive, not a
replacement — worth knowing before telling someone to re-pass their whole config.

### `ultracode` is a settings key, not an `--effort` value

🔬 `--effort` is registered with `(low, medium, high, xhigh, max)`. `ultracode` is not among
them. Its own strings say where it lives:

```
Whether ultracode (xhigh effort plus standing dynamic-workflow orchestration) is active for
the session. Set per session via the `ultracode` settings key (--settings or
apply_flag_settings).

- ultracode: xhigh + dynamic workflow orchestration (this session only)

apply_flag_settings: ultracode is not available for this session (dynamic workflows are off,
or the model / your organization does not allow xhigh effort)
```

So the spelling is `--settings '{"ultracode":true}'`, and it is scoped to one session by
design. 🔬 A sibling string shows Remote Control may change only two effort-related keys:
`cannot be changed over Remote Control (only effortLevel and ultracode can)`.

### `--betas` has a one-entry allowlist

🔬 Un-minified from the binary:

```js
// The beta descriptor factory: every beta is {name, header}.
function beta(name, header) { return Object.freeze({ name, header }) }

const LONG_CONTEXT = beta("long_context", "context-1m-2025-08-07")

// The ENTIRE allowlist for user-supplied betas — one member.
const USER_SETTABLE_BETAS = new Set([LONG_CONTEXT])

// Validates --betas / ANTHROPIC_BETAS.
function validateUserBetas(requested) {                             // was: e
  if (!requested || requested.length === 0) return
  if (isNotApiKeyUser()) {
    console.warn("Warning: Custom betas are only available for API key users. Ignoring provided betas.")
    return
  }
  const { allowed, disallowed } = partition(requested)
  for (const beta of disallowed)
    console.warn(`Warning: Beta header '${beta}' is not allowed. `
               + `Only the following betas are supported: ${[...USER_SETTABLE_BETAS].join(", ")}`)
  return allowed.length > 0 ? allowed : undefined
}
```

Two gates, either of which is fatal for a subscription user: API-key-only, and an allowlist
holding just `context-1m-2025-08-07`. 🔬 The binary contains ~30 other beta header strings
(`effort-2025-11-24`, `advanced-tool-use-2025-11-20`, `structured-outputs-2025-12-15`,
`agent-memory-2026-07-22`, `interleaved-thinking-2025-05-14`, `context-management-2025-06-27`,
`per-turn-control-2026-07-01`, …) but Claude Code selects those per model itself. They cannot
be forced from the command line.

`ANTHROPIC_BETAS` is the env-var form and is appended through the same path.

### IDE auto-connect: `--ide` is usually redundant

🔬 Un-minified:

```js
// Decides whether to attach to an IDE at startup.
function shouldAutoConnectIde(ideFlag = false) {                    // was: e
  // An explicit false env var is an absolute veto.
  if (env.CLAUDE_CODE_AUTO_CONNECT_IDE === false) return false
  return Boolean(
    config().autoConnectIde                 // the /config toggle
    || ideFlag                              // the --ide flag
    || isSupportedIdeTerminal()             // running INSIDE VS Code/JetBrains
    || env.CLAUDE_CODE_SSE_PORT !== undefined
    || env.CLAUDE_CODE_AUTO_CONNECT_IDE === true
  )
}
```

The `isSupportedIdeTerminal()` branch means a session launched from an IDE's integrated
terminal connects with no flag at all. `--ide` only matters from an outside terminal (iTerm,
Ghostty, tmux). 📖 CHANGELOG 5588 added `CLAUDE_CODE_AUTO_CONNECT_IDE=false` as the opt-out,
which confirms auto-connect is the default in that case.

### Bash auto-backgrounding, and the variable that actually caps it

📖 A long-standing entry: *"Auto-background long-running bash commands instead of killing
them. Customize with `BASH_DEFAULT_TIMEOUT_MS`"*. 🔬 The constants:

```js
const DEFAULT_BASH_TIMEOUT_MS = 120_000   // 2 min
const MAX_BASH_TIMEOUT_MS     = 600_000   // 10 min
const MIN_AUTO_BACKGROUND_MS  = 2_000

function defaultBashTimeout(env = process.env) {       // was: aye
  const raw = env.BASH_DEFAULT_TIMEOUT_MS
  if (raw) { const n = parseInt(raw); if (!isNaN(n) && n > 0) return n }
  return DEFAULT_BASH_TIMEOUT_MS
}

function maxBashTimeout(env = process.env) {
  const raw = env.BASH_MAX_TIMEOUT_MS
  if (raw) { const n = parseInt(raw); if (!isNaN(n) && n > 0) return Math.max(n, defaultBashTimeout(env)) }
  return Math.max(MAX_BASH_TIMEOUT_MS, defaultBashTimeout(env))
}
```

⚠️ `BASH_DEFAULT_TIMEOUT_MS` sets only the **default**; the model may still request up to
`maxBashTimeout()` per call. The variable that imposes a ceiling is not in `--help`:

```js
// Clamps a per-call timeout so a stalled command backgrounds sooner.
function effectiveBashTimeout({ requestedTimeoutMs, isMainAgent, canAutoBackground, env = process.env }) {
  // Subagents and non-backgroundable calls are exempt.
  if (!isMainAgent || !canAutoBackground) return requestedTimeoutMs
  const raw = env.CLAUDE_CODE_AUTO_BACKGROUND_TIMEOUT_MS
  if (!raw) return requestedTimeoutMs
  const configured = parseInt(raw)
  if (isNaN(configured) || configured <= 0) return requestedTimeoutMs
  // Never below the 2s floor, never above what the caller asked for.
  return Math.min(requestedTimeoutMs, Math.max(configured, MIN_AUTO_BACKGROUND_MS))
}
```

So `CLAUDE_CODE_AUTO_BACKGROUND_TIMEOUT_MS=45000` forces every foreground Bash call in the
main agent to background after 45 s, overriding a larger `timeout` the model passed. This is
the knob for "let Claude keep thinking while a stalled command runs" — `BASH_DEFAULT_TIMEOUT_MS`
is not, because it is overridable per call.

🔬 Related names, all present in the binary:

| Env var | Effect |
|---|---|
| `CLAUDE_CODE_AUTO_BACKGROUND_TIMEOUT_MS` | Hard ceiling on main-agent Bash timeout; floor 2000 ms |
| `CLAUDE_CODE_MCP_AUTO_BACKGROUND_MS` | 📖 Same for MCP tool calls; default 2 min |
| `CLAUDE_AUTO_BACKGROUND_TASKS` | Gates the worker check-in path below |
| `CLAUDE_CODE_AUTO_BACKGROUND_WORKER_CHECKIN_SECONDS` | Check-in cadence; read only when the above is set |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | 📖 Disables auto-backgrounding and Ctrl+B entirely |

🔬 The check-in resolver, showing the dependency:

```js
function workerCheckinMs() {                          // was: oWt
  let seconds
  if (isCoordinator()) seconds = env.CLAUDE_CODE_COORDINATOR_WORKER_CHECKIN_SECONDS
  else if (env.CLAUDE_AUTO_BACKGROUND_TASKS) seconds = env.CLAUDE_CODE_AUTO_BACKGROUND_WORKER_CHECKIN_SECONDS
  return seconds === undefined ? undefined : seconds * 1000
}
```

### Claude in Chrome: the env var outranks the settings key

This extends the 2.1.246 section above, which established the gate order on Linux. 🔬 The
same function in 2.1.251, un-minified:

```js
// Called with the parsed --chrome/--no-chrome value.
function chromeEnabled(chromeFlag) {                  // was: e
  // Hard block: the OAuth token must carry an accepted scope.
  if (!hasAcceptedOAuthScope()) {
    log("[Claude in Chrome] Disabled: OAuth token has no scope accepted by "
      + "/api/oauth/validate (needs user:profile, user:office, or user:ccr_inference; "
      + "env-var and setup-token sessions default to user:inference only)")
    return false
  }
  if (chromeFlag === true)  return true               // --chrome
  if (chromeFlag === false) return false              // --no-chrome
  if (env.CLAUDE_CODE_ENABLE_CFC === true)  return true
  if (env.CLAUDE_CODE_ENABLE_CFC === false) return false
  if (isOtherwiseBlocked()) return false
  const cfg = readConfig()
  if (cfg.claudeInChromeDefaultEnabled !== undefined) return cfg.claudeInChromeDefaultEnabled
  return false                                        // DEFAULT: OFF
}
```

⚠️ The important part is **downstream** of that function. Two later wiring gates read the env
var but not the settings key:

```js
// enterprise policy block
const blockedByEnterprise =
  chromeConfigured && flags.chrome !== true
  && env.CLAUDE_CODE_ENABLE_CFC !== true
  && mcpServerDenied
// → "[Claude in Chrome] Skipping chrome wiring: blocked by enterprise MCP config
//    or managed deniedMcpServers policy"

// safe/restricted mode block
const blockedBySafeMode =
  chromeConfigured && flags.chrome !== true
  && (env.CLAUDE_CODE_ENABLE_CFC !== true && isSafeMode() || isRestricted())
// → "[Claude in Chrome] Skipping chrome wiring: --safe-mode or --restricted"
```

So `CLAUDE_CODE_ENABLE_CFC=1` and `--chrome` can punch through an enterprise
`deniedMcpServers` policy and `--safe-mode`; `claudeInChromeDefaultEnabled: true` cannot.
The two are **not** interchangeable, even though `chromeEnabled()` alone makes them look it.
`--restricted` is absolute — no env var clears it.

🔬 `claudeInChromeDefaultEnabled` is a real config key (it appears in the config key list
beside `inputNeededNotifEnabled` and `agentPushNotifEnabled`) and is surfaced in `/config` as
`"Claude in Chrome enabled by default"`.

### Background-session subcommands

🧪 `--help` for each, quoted exactly — these are newly listed in `claude --help` as of 2.1.251:

```
claude attach <id>   Open the background session in this terminal. ← returns to agent view,
                     Ctrl+Z drops back to your shell. The session keeps running either way.
claude logs <id>     Print the background session's recent terminal output.
claude respawn <id>|--all
                     Restart a background session (or all of them) so it picks up the
                     current Claude binary.
claude stop <id>     Stop a background session. Its conversation is kept; resume it later
                     with `claude attach <id>`.
claude rm <id>       Delete a background session and its worktree. Unlike `stop`, works on
                     already-exited sessions.
```

🔬 `claude agents` accepts `--add-dir`, `--agent`, `--effort`, `--mcp-config`, `--model`,
`--permission-mode`, `--plugin-dir`, `--restricted`, `--setting-sources`, `--settings`,
`--strict-mcp-config`, and `--dangerously-skip-permissions` as **defaults for every session it
dispatches**, plus `--json` to print the list without a TTY.

### Subagent-related env vars

🔬 Present in the binary, all confirmed by name:

| Env var | Effect |
|---|---|
| `CLAUDE_CODE_FORK_SUBAGENT` | 📖 Forked subagents on external builds; works in `-p`/SDK since 2.1.x |
| `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` | 📖 Nesting depth; default 3, set 1 to disable nesting |
| `CLAUDE_CODE_SUBAGENT_MODEL` | 📖 As of 2.1.251 this is a *default*, not an override — an agent definition's `model:` and an explicit per-spawn model now win |
| `CLAUDE_CODE_MAX_SUBAGENTS_PER_SESSION` | Session-wide subagent cap |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | 📖 One implicit team per session; spawn teammates via the Agent tool's `name` |

### Effort is stored per model

🔬 `modelSettings` holds a per-model `effortLevel` that overrides the top-level one:

```json
"effortLevel": "high",
"modelSettings": {
  "claude-fable-5": { "effortLevel": "xhigh" },
  "claude-opus-5":  { "effortLevel": "xhigh" }
}
```

📖 2.1.251: *"Changed `/effort` to save your default effort level per model, so each model
keeps its own setting when you switch."* A top-level `effortLevel` alone therefore silently
applies to every model **except** those named in `modelSettings`.

🔬 Also in 2.1.251: *"Fixed Opus 5 requests failing with 'effort … is not supported when
thinking is disabled' when effort was xhigh/max and thinking was turned off; effort is now
sent as `high` in that case."* So `MAX_THINKING_TOKENS=0` quietly downgrades a `max` effort
setting.

### Method note

`strings` on the 197 MB binary is slow to re-run per query. Dump it once, then use `rg` with
`--only-matching` and fixed-width context windows to read around a symbol:

```sh
BIN=~/.local/share/claude/versions/2.1.251
strings -a "$BIN" > /tmp/claude-strings.txt

# read 260 chars before and 400 after a name, to catch the enclosing function
rg --only-matching '.{260}CLAUDE_CODE_AUTO_BACKGROUND_TIMEOUT_MS.{400}' /tmp/claude-strings.txt

# the env-var registry is one giant object literal; this enumerates it
rg --only-matching 'CLAUDE_CODE_[A-Z0-9_]+:\(\)=>' /tmp/claude-strings.txt | sort --unique
```

---

## 2.1.252

Recorded **2026-08-31** on the ByteDance work MacBook, running 2.1.252. The question that
produced this section: *how can the "Context is 97% full" suggestions say one tool is using
2.7m tokens — 272% of context?*

### Context suggestions can claim >100% because they count base64 as text 🧪📦🔬

🧪 Observed in a claude-in-chrome-heavy session whose real live context was 966k of 1M
(97% — that number comes from the API's own `usage` block and is exact). The suggestions
panel under the warning showed:

```
mcp__claude-in-chrome__browser_batch using 2.7m tokens (272%) → save ~543k
mcp__claude-in-chrome__computer using 623.1k tokens (62%) → save ~124.6k
```

2.7m ÷ 1M = 272%: the denominator is the raw context window, and the numerator is an
estimate that can exceed it by multiples.

📦 The pipeline, from `src/` (mechanism only; the 🔬 strings below confirm the feature is in
the shipped binary):

- `src/utils/contextSuggestions.ts` — the generic per-tool case fires at ≥20%:
  `` `${toolName} using ${tokenStr} tokens (${percent.toFixed(0)}%)` `` with
  `percent = (tokens / data.rawMaxTokens) * 100` and `savingsTokens = tokens * 0.2`
  (Grep and WebFetch get dedicated cases with 0.3 / 0.4 savings factors).
- The per-tool `tokens` comes from `approximateMessageTokens()` in
  `src/utils/analyzeContext.ts`. It walks the **microcompacted live messages** — so this is
  the live conversation, not a whole-transcript sum — and for every content block does:

  ```ts
  const blockStr = jsonStringify(block)
  const blockTokens = roughTokenCountEstimation(blockStr) // = blockStr.length / 4
  ```

  `tool_result` blocks are attributed to a tool through a `tool_use_id → name` map.
- The failure: a `tool_result` holding a base64 screenshot gets the base64 counted at 4
  bytes/token. A ~1 MB PNG → ~1.37 MB base64 → **~340k estimated tokens** for an image the
  API bills at ~1.5k (vision patch formula; capped at 4,784 even on the high-res tier).
  That is a ~200× overcount per screenshot, so a handful of live screenshots inside
  `browser_batch` results adds up to "2.7m tokens".
- The kicker: the correct estimator already exists in the same codebase.
  `roughTokenCountEstimationForBlock()` in `src/services/tokenEstimation.ts` special-cases
  `image` and `document` blocks at a **flat 2000 tokens**, and its comment describes this
  exact failure: *"base64 PDF in source.data. Must NOT reach the jsonStringify catch-all — a
  1MB PDF is ~1.33M base64 chars → ~325k estimated tokens, vs the ~2000 the API actually
  charges."* The suggestions breakdown does not use it.

🔬 `strings` on `~/.local/share/claude/versions/2.1.252` finds
`This tool is consuming a significant portion of context.`,
`Autocompact will trigger soon, which discards older messages. Use /compact now to control what gets kept.`,
and the identifier `rawMaxTokens`. The feature and its window-relative denominator are in the
shipped binary; the estimation path itself was not decompiled, so the 4-bytes/token mechanism
stays a 📦 claim — the observed 272% is its fingerprint.

📖 Nothing in `CHANGELOG.md` through 2.1.252 mentions fixing suggestion token estimates.

⇒ Practical reading: **trust the top-line context percentage; distrust the per-tool
suggestion numbers whenever a tool returns images**, and discount their "save ~N" the same
way. Reported via `/feedback`, receipt `4980a14a-06df-4796-aa24-88249cb0e032`.

---

## 2.1.259

Recorded **2026-09-03** on the Windows gaming PC (`windows-pc`).

```
Running: native (2.1.259)
Commit:  9b549c8d1c72
Platform: win32-x64
Path:    C:\Users\Max\.local\bin\claude.exe
```

The question that produced this section: *how do I make the `!` prefix run PowerShell instead
of Git Bash on Windows, from a `settings.json` checked into dotfiles shared with a Mac and two
Linux boxes?* Every load-bearing answer turned out to be **binary-only** — this is the clearest
case yet of the snapshot describing a mechanism that has since grown a second half.

### The setting 📦🔬

`defaultShell` is in both, at `src/utils/settings/types.ts:464`. 🔬 Its `describe()` string is
verbatim in the binary:

> Default shell for input-box `!` commands. Defaults to `'bash'` on all platforms (no Windows
> auto-flip).

Enum: `bash | powershell`. It governs the **input box only** — never which shell tool the model
picks. 📦 Two sibling files say so out loud: `frontmatterParser.ts:55` ("Never consults
settings.defaultShell: skills are portable across platforms") and `promptShellExecution.ts:65`.

### Binary-only: `resolveDefaultShell` has grown a self-correcting flip 🔬

📦 The snapshot is one line with no fallback:

```ts
export function resolveDefaultShell(): 'bash' | 'powershell' {
  return getInitialSettings().defaultShell ?? 'bash'
}
```

🔬 2.1.259 is not. *Un-minified.* `Je()` is the settings
getter, `cs()` and `lH()` are covered below.

```js
function resolveDefaultShell() {                      // was: iat
  const setting = getSettings().defaultShell
  if (setting === "bash"       && !bashAvailable())  return "powershell"
  if (setting === "powershell" && !psToolEnabled())  return "bash"
  return setting ?? (bashAvailable() ? "bash" : "powershell")
}
```

<details><summary>Original minified form, as it appears in the binary</summary>

```js
function iat(){let e=Je().defaultShell;if(e==="bash"&&!cs())return"powershell";if(e==="powershell"&&!lH())return"bash";return e??(cs()?"bash":"powershell")}
export{iat};
```

</details>

⇒ **This flip is what makes one checked-in key portable.** Set `"defaultShell": "powershell"` in
a shared dotfile and macOS/Linux revert to `bash` on their own, because the PowerShell gate below
is false there. No per-machine settings file, and none exists at user scope anyway — 📦
`settings.ts:305` resolves `localSettings` to a **project**-relative `.claude/settings.local.json`.

### Binary-only: the PowerShell-tool gate, and a rollout flag named `tengu_cobalt_ridge` 🔬

📦 The snapshot gates on the env var alone, and is unconditionally false off Windows:

```ts
export function isPowerShellToolEnabled(): boolean {
  if (getPlatform() !== 'windows') return false
  return process.env.USER_TYPE === 'ant'
    ? !isEnvDefinedFalsy(process.env.CLAUDE_CODE_USE_POWERSHELL_TOOL)
    : isEnvTruthy(process.env.CLAUDE_CODE_USE_POWERSHELL_TOOL)
}
```

🔬 2.1.259 has replaced the `USER_TYPE` branch with a rollout flag, and — the part that matters —
**no longer returns false off Windows**:

```js
function psToolEnabled() {                            // was: lH
  const envVar = parsedEnv.CLAUDE_CODE_USE_POWERSHELL_TOOL
  if (getPlatform() !== "windows") return envVar === true
  if (envVar !== undefined) return envVar                    // explicit env var wins
  if (findGitBash() === null) return true                    // no bash ⇒ PS tool on
  return featureFlag("tengu_cobalt_ridge", false)            // gradual rollout
}

function bashAvailable() {                            // was: cs
  if (getPlatform() !== "windows") return true
  return findGitBash() !== null
}

function defaultHookShell() {                         // was: OM
  return bashAvailable() ? "bash" : "powershell"
}
```

<details><summary>Original minified form, as it appears in the binary</summary>

```js
function lH(){let e=a.CLAUDE_CODE_USE_POWERSHELL_TOOL;if(D()!=="windows")return e===!0;if(e!==void 0)return e;if(q$()===null)return!0;return P("tengu_cobalt_ridge",!1)}function cs(){if(D()!=="windows")return!0;return q$()!==null}function OM(){return cs()?"bash":"powershell"}
```

</details>

`q$()` is not resolved by name; it is read as "locate Git Bash, or null" from its two uses.

Three things fall out, none of them in the snapshot:

1. 🧪 **`tengu_cobalt_ridge` is why the tool appears with no env var set.** On this machine
   `$env:CLAUDE_CODE_USE_POWERSHELL_TOOL` and `$env:USER_TYPE` are both empty and Git Bash is
   installed, yet the session had the PowerShell tool. Only the flag branch can produce that.
   ⇒ This retro-explains the loose end in an earlier session, which recorded the tool arriving
   "by another route (rollout or settings)". 🔬 The startup tip is still env-var-only —
   `isRelevant: process.env.CLAUDE_CODE_USE_POWERSHELL_TOOL === undefined` — so it keeps firing
   at people who already have the tool.
2. **No Git Bash ⇒ the PowerShell tool turns itself on.** A Windows box without Git for Windows
   needs no configuration at all.
3. ⚠️ **`CLAUDE_CODE_USE_POWERSHELL_TOOL` is now live on macOS and Linux.** The snapshot returns
   `false` there whatever the env var says; the binary returns `envVar === true`. So putting that
   variable in a shared `settings.env` block would make `resolveDefaultShell()` answer
   `"powershell"` on a Mac and send `!` at a `pwsh` that may not exist. **Set `defaultShell` in
   shared dotfiles; never the env var.**

### Binary-only: what `!` actually spawns 🔬

The `!` handler, and the child-process shape. Both callers of `resolveDefaultShell` are here:

```js
async function handleBangCommand(input, …) {
  const usePowerShell = psToolEnabled() && resolveDefaultShell() === "powershell"
  const respond = getSettings().respondToBashCommands ?? true
  logEvent("tengu_input_bash", { powershell: usePowerShell, respond })
  …
}

// the spawn helper
const { file, args } = resolveDefaultShell() === "powershell"
  ? { file: "pwsh", args: ["-NoProfile", "-Command", command] }
  : { file: "/bin/sh", args: [ … ] }
```

<details><summary>Original minified form, as it appears in the binary</summary>

```js
async function F(n,b,e){let u=lH()&&iat()==="powershell",l=Je().respondToBashCommands??!0;s("tengu_input_bash",{powershell:u,respond:l});…
async function b(e){let{command:s}=e,i=e.cwd??ne(),{file:a,args:n}=iat()==="powershell"?{file:"pwsh",args:["-NoProfile","-Command",s]}:{file:"/bin/sh",a…
```

</details>

- 🔬 **`pwsh` is hardcoded** — PowerShell 7. There is no `powershell.exe` 5.1 fallback.
- 🔬 **`-NoProfile`** — the user's PowerShell profile does not load, so profile aliases and
  functions do not exist in `!` commands.
- 🔬 `respondToBashCommands` is **binary-only**: absent from the snapshot, and absent from the
  snapshot's `logEvent`, which sends `{ powershell }` alone. Its schema string reads *"Whether
  Claude responds after an input-box ! bash command runs. Set to false to add the command output
  to context without a response."*
- 🔬 Adjacent, unrelated, and easy to confuse with `defaultShell`: `CLAUDE_CODE_SHELL` (with
  `Using shell override:` and `" is not a valid bash/zsh path, falling back to detection`) and
  `CLAUDE_CODE_SHELL_PREFIX`. Those pick the POSIX login shell, not the `!` interpreter.

### Binary-only: auto and bypass modes inject a *use-Bash* instruction 🔬

Not in the snapshot — `rg 'While auto mode is active' src/` returns nothing. The binary's string
table holds a template whose tool name is interpolated:

```
Do your work through the ⟨tool⟩ tool wherever it can accomplish the job: read files with cat,
head, or sed -n, search with grep and find, and make file changes with sed, heredocs, or short
scripts, rather than using the dedicated ⟨tools⟩. Fall back to a dedicated tool only when
⟨tool⟩ genuinely cannot do the job.
```

🔬 Two sibling headers sit beside it: `While bypass permissions mode is active:` and
`While auto mode is active:`. 🧪 Both render with **Bash**, and the command list is POSIX.

⇒ On Windows this instruction actively pushes the model at Git Bash. A `CLAUDE.md` line telling
Claude to prefer PowerShell is therefore a counterweight to a *harness* instruction, not to a
model habit — and `defaultShell` does nothing to help, because it is never read on the
tool-selection path.

### 📦 Settings are parsed with plain `JSON.parse` — comments void the file

Worth recording because the failure is silent. `parseSettingsFile` → `safeParseJSON`
(`src/utils/json.ts`) → `JSON.parse(stripBOM(json))`, returning **`null`** on any error. One `//`
line therefore discards the entire settings file with no message. `safeParseJSONC` exists in the
same module but is reserved for VS Code keybindings.

The outer settings object is `.passthrough()` (`types.ts:1072`), so an unknown key is preserved
and ignored. ⇒ The only safe way to comment a `settings.json` is a sibling key, e.g.
`"// defaultShell": "…"`.

### Practical summary

| Want | Do |
|---|---|
| `!` runs PowerShell on Windows only | `"defaultShell": "powershell"` — safe in shared dotfiles |
| The same on every OS | also set `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` (and accept `pwsh` must exist) |
| Claude to *choose* PowerShell | a `CLAUDE.md` instruction; no setting does this |
| A comment in `settings.json` | a sibling `"// key"`, never `//` |
