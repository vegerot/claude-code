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

const tmuxVersionMatch = tmuxVersionResult.stdout.match(/(\d+)\.(\d+)/)  // was: m
const major = Number(tmuxVersionMatch?.[1])
const minor = Number(tmuxVersionMatch?.[2])

// tmux gained `-e KEY=VALUE` on new-session in 3.2, so gate on >= 3.2.
const supportsEnvFlag =                                                   // was: h
  tmuxVersionMatch !== null && (major > 3 || (major === 3 && minor >= 2))

// Modern tmux: hand the vars over as explicit -e arguments.
const envFlagArgs = supportsEnvFlag                                       // was: g
  ? Object.entries(tmuxEnv).flatMap(([key, value]) => ["-e", `${key}=${value}`])
  : []

// Old tmux: fall back to leaking them through the child's environment.
const childEnv = { ...process.env, ...(supportsEnvFlag ? {} : tmuxEnv) }  // was: y
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
const SSH_TRANSPORT_ENV_VAR = 'CLAUDE_DESKTOP_SSH_TRANSPORT'    // was: t

// The env override is honored ONLY on internal ("Nest") or dev builds.
function sshTransportOverrideFromEnv() {                        // was: n
  if (!isInternalBuild() && !isDevBuild()) return undefined     // was: e.nC(), e.eC()
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
