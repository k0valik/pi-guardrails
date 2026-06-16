## [AlainS7](https://github.com/k0valik/pi-guardrails/compare/main...AlainS7:pi-guardrails:main)

**Branch:** `main` · **Status:** diverged · **Commits:** 14 · **Files:** 13

**Overview:** The AlainS7 fork adds four feature areas: (1) glob pattern support + global-scope grants,
(2) symlink-aware boundary checks, (3) extension edit gate (blocks edits to `~/.pi/agent/extensions/`
without approval), and (4) read-only bash command detection. Also includes ctx7 library ID filtering
for bash path extraction.

**⚠️ Architecture mismatch:** This fork is based on the **old monolithic architecture** (`src/hooks/`, `src/utils/`).
Our repo has been restructured into `src/core/`, `src/shared/`, `extensions/`. File paths don't match -
direct cherry-picking will fail. Any port would be manual.

---

### Feature Area 1: Glob pattern support + global-scope grants

**Commits:** `29544218` (path access grants & persistence)

**Files changed:**
- `src/utils/path-access.ts` (+76/-3) - `isPathAllowed` gets wildcard compat → `/*` and `/**` entries treated as directory grants; optional glob matching via `matchesGlob`
- `src/hooks/path-access.ts` (+127/-8) - grant persistence scope `"local"` → `"global"`
- `src/utils/path-access.test.ts` (+36) - tests for wildcard forms

**What it does:**
Allows users to use `allowedPaths` entries like `/opt/homebrew/lib/node_modules/*` or `/opt/homebrew/lib/node_modules/**` as directory grants (backward compat). Also changes where "always" grants are persisted - from project-local config to global config.

**Key code changes:**
- `isPathAllowed`: adds early `continue` for directory entries; wildcard suffix normalization; `matchesGlob` integration
- Hook: `checkPathAccess` → `checkPathAccessResolved` (async, symlink-aware variant)
- Grant persistence: `"local"` → `"global"` scope for `allow-file-always` / `allow-dir-always`

**Our equivalent:**
- Our `src/core/paths/access.ts` has the simpler `isPathAllowed` without glob compat
- Our `extensions/path-access/grants.ts` uses `"local"` scope - need to understand if `"global"` maps to our config

**Assessment:** Useful feature, especially the wildcard compat (`/*`, `/**` → directory grant). The `"global"` scope change is a design choice - our `"local"` scope persists to project config; `"global"` would persist to user-level config. Worth porting if we want this flexibility.

**Verdict:** ⏳ Maybe - glob compat is worth porting; scope change needs review

---

### Feature Area 2: Symlink-aware boundary checks

**Commits:** `64251239` · `ce4110c7` · `8b39b80b`

**Files changed:**
- `src/utils/path.ts` (+50/-1) - adds `resolvePathWithSymlinks` and `isWithinBoundaryResolved`
- `src/utils/path.test.ts` (+24) - integration tests for symlink escape detection
- `src/utils/path-access.ts` - adds `checkPathAccessResolved` (async, uses `isWithinBoundaryResolved`)

**What it does:**
Adds symlink-aware path boundary checking. `resolvePathWithSymlinks` walks the path from leaf upward,
resolving each segment through `realpath` until one succeeds. `isWithinBoundaryResolved` resolves both
target and root before doing the lexical boundary check. Prevents symlink escape attacks (e.g., a symlink
inside an allowed dir pointing outside).

**Key code changes:**
- `resolvePathWithSymlinks`: iterative algorithm - starts at the full path, tries `realpath`, if it fails, moves up one segment and tracks the unresolved tail
- `isWithinBoundaryResolved`: async, resolves both paths in parallel, then calls lexical `isWithinBoundary`

**Our equivalent:**
- Our `src/core/paths/path.ts` has only the lexical `isWithinBoundary` - no symlink resolution
- Our `src/core/paths/access.ts` has synchronous `checkPathAccess` - no async/resolved variant
- Our `extensions/path-access/rules.ts` creates rules using `checkPathAccess`

**Assessment:** Valuable security hardening. The `resolvePathWithSymlinks` algorithm is well-designed
(handles non-existent leaf segments by walking up). However, introducing async I/O into the hot path
has latency implications. Could be integrated as an optional hardening layer.

**Verdict:** ✅ Take - security improvement, but needs manual porting to `src/core/paths/path.ts` and `src/core/paths/access.ts`

---

### Feature Area 3: Extension edit gate

**Commits:** `3448e09b` · `91489af3` · `99183520`

**Files changed:**
- `src/hooks/path-access.ts` (+127/-8) - adds `enforceExtensionsEditBoundary`, `enforceExtensionsBashBoundary`, `sessionEditApproved`
- `src/hooks/path-access.test.ts` (+460) - tests for extension edit gating

**What it does:**
Blocks `write`/`edit` tool calls targeting `~/.pi/agent/extensions/` without explicit user approval.
Bash commands mentioning `.pi/agent/extensions` are also gated (unless clearly read-only).
Session approval is tracked via `sessionEditApproved` Set keyed by session ID.
In headless mode (no UI), extensions edits are denied by default.

**Key code changes:**
- Constants: `GLOBAL_EXTENSIONS_ROOT`, `EDIT_TOOLS` (`write`, `edit`), `EDIT_SELECTIONS`
- `sessionEditApproved`: `Set<string>` tracking approved session IDs
- `enforceExtensionsEditBoundary`: checks if tool+path targets extensions root via `isWithinBoundaryResolved`
- `enforceExtensionsBashBoundary`: checks if bash command mentions extension path markers
- `requestExtensionsEditApproval`: prompts user with Allow-once / Allow-session / Deny
- Hooked into both the file-per-target loop AND the bash pre-processing in `setupPathAccessHook`

**Our equivalent:**
- Nothing equivalent exists. Our `extensions/guardrails/hooks/` could be extended with a similar gate.
- The policy-based architecture allows a dedicated feature extension.

**Assessment:** Very useful safety feature, especially for an agent that might be tricked into
modifying its own extensions. Would fit naturally as a policy rule or separate feature extension.
Depends on Feature 2 (uses `isWithinBoundaryResolved`).

**Verdict:** ✅ Take - high-value safety feature. Manual port to a policy rule in `extensions/guardrails/hooks/`.

---

### Feature Area 4: Read-only bash command detection

**Commits:** `41728fbb` · `5dfb89d1` · `25a3a988` · `2e596ce1`

**Files changed:**
- `src/utils/bash-intent.ts` (added, +183) - `isClearlyReadOnlyBashCommand` function
- `src/utils/bash-intent.test.ts` (added, +56) - tests
- `src/hooks/policies.ts` (+10/-1) - skip read-only bash commands for `readOnly` policy protection
- `src/hooks/policies.test.ts` (added, +160) - integration tests for readOnly + bash behavior
- `README.md` (+11/-1) - update docs

**What it does:**
Skips policy enforcement for `readOnly`-protected paths when the bash command is clearly read-only
(e.g., `ls`, `cat`, `rg`, `head`, `find` without destructive flags).
Uses AST parsing via `@aliou/sh` to detect mutating operations (redirections >, >>, pipe-to-write,
shell substitutions, path-qualified binaries).

**Key code changes:**
- `isClearlyReadOnlyBashCommand`:
  - Allowlist of read-only commands: `ls`, `rg`, `grep`, `cat`, `head`, `tail`, `wc`, `stat`, `file`, `find`, `fd`, `tree`, `du`, `sort`, `uniq`, `cut`, `pwd`, `realpath`, `readlink`, ...
  - Danger signals (return false): output redirections (`>`), shell substitutions (`$()`, backticks), path-qualified binaries (`/tmp/ls`), `&&` chaining with mutations
- In `setupPoliciesHook`: before blocking a readOnly-bash + tool, checks `isClearlyReadOnlyBashCommand` and continues if true

**Our equivalent:**
- Nothing equivalent exists. Our `src/core/shell/command-args.ts` has command-specific arg classification but no read-only intent detection.
- Our `extensions/guardrails/hooks/policies.ts` would need a similar bypass.

**Assessment:** Reduces noise for safe read operations on readOnly-protected paths. The implementation
is conservative (uses a command allowlist + danger signal checks).
Our `classifyCommandArgs` infrastructure could be extended or the simpler allowlist approach used.

**Verdict:** ⏳ Maybe - useful quality-of-life improvement, moderate priority

---

### Side feature: ctx7 library ID filtering

**Commits:** `1ac24103` · `a09f3e8c` · `43d5eb54`

**Files changed:**
- `src/utils/bash-paths.ts` (+31/-11) - adds `isCtx7Invocation`, `isCtx7LibraryId`, skip logic in arg processing
- `src/utils/bash-paths.test.ts` (+29) - tests

**What it does:**
Prevents ctx7 library IDs like `/badlogic/pi-mono` or `/vercel/next.js/v14.3.0` (which look like
Unix paths but are library identifiers) from being treated as file-system paths. Detects ctx7
invocations and skips positional args after `docs` that match the library ID pattern (2-4 segments
of alphanumeric/dot/hyphen/underscore characters).

**Key code changes:**
- `isCtx7Invocation`: checks if any word matches `/(^|[\/])ctx7(@|$)/`
- `isCtx7LibraryId`: rejects if segment count outside [2,4] or non-alphanumeric segments
- Added `skip` parameter to `addCandidate` - when true, the token is silently dropped

**Our equivalent:**
- Our `src/shared/paths/bash-paths.ts` uses `classifyCommandArgs` from `src/core/shell/command-args.ts`
- We could add ctx7 awareness to `classifyCommandArgs` instead

**Assessment:** Very narrow fix for one specific tool invocation pattern. Less relevant given our
codebase already has `classifyCommandArgs` which is the right place for tool-specific arg handling.

**Verdict:** ❌ Skip - too narrow; if needed, do it via `classifyCommandArgs`

---

### Summary table

| Feature | Verdict | Effort | Notes |
|---|---|---|---|
| Glob compat in `isPathAllowed` | ⏳ Maybe | Low | Simple port to `src/core/paths/access.ts` |
| Grant scope `local` → `global` | ⏳ Maybe | Low | Needs config architecture review |
| Symlink-aware boundary check | ✅ Take | Medium | Port to `src/core/paths/path.ts` + `access.ts` |
| Extension edit gate | ✅ Take | Medium | New policy rule or feature extension |
| Read-only bash detection | ⏳ Maybe | Medium | New util + hook bypass in policies |
| ctx7 library ID filtering | ❌ Skip | Low | Do via `classifyCommandArgs` if needed |

**Bottom line:** 2 features worth manual porting (symlink-aware checks + extension edit gate).
The glob compat and read-only bash detection are useful but lower priority.
All ports are manual - the architecture mismatch makes direct cherry-picking impossible.

---

## [jeprecated](https://github.com/k0valik/pi-guardrails/compare/main...jeprecated:pi-guardrails:main)

**Branch:** `main` · **Status:** diverged · **Commits:** 2 · **Files:** 22

**Overview:** The jeprecated fork adds an **approval broker routing** system - a generic, configurable
approval routing layer that decouples approval prompts from specific UIs. Allows routing approval
requests through multiple sources (local UI, remote CLI) with configurable strategies (first-terminal,
all, threshold, veto, etc.). Also includes `agent-tick` CLI integration for remote human approval.

**⚠️ Architecture mismatch:** Same as AlainS7 - based on the old monolithic structure (`src/hooks/`,
`src/approval/`). 22 files changed, ~3,500 lines added. Direct cherry-picking impossible.

---

### Feature: Approval Broker Routing

**Commits:** `2409f420` (main feature) · `62870ce0` (agent-tick CLI fix)

**Files changed:**

New files under `src/approval/`:
- `types.ts` (+157) - core types: `ApprovalRequest`, `ApprovalDecision`, `ApprovalSource`, `ApprovalStrategy`, `ApprovalBrokerEvent`, `ApprovalBrokerResult`
- `strategy.ts` (+122) - strategy resolution with presets (first-terminal, all, threshold, any-approve, veto-threshold)
- `broker.ts` (+561) - main `ApprovalBroker` class: manages source startup, collects decisions, evaluates strategy, handles timeouts/cancellation
- `request.ts` (+68) - request creation, correlation tokens, action fingerprinting
- `local-source.ts` (+135) - `createLocalApprovalSource`: wraps Pi UI prompts as an `ApprovalSource`
- `agent-tick-source.ts` (+599) - `createAgentTickApprovalSource`: integrates with external `agent-tick` CLI for remote approval
- `source-factory.ts` (+89) - `buildApprovalRouteSources`: builds source list from config routes
- `index.ts` (+55) - barrel exports
- Various `.test.ts` files (+1,030)

Modified files:
- `src/config.ts` (+99) - adds `ApprovalBrokerConfig` with sources, routes, strategies, defaults
- `src/hooks/path-access.ts` (+222/-53) - brokers path access prompts through the approval broker
- `src/hooks/permission-gate/index.ts` (+99/-23) - brokers permission gate prompts through the approval broker
- `src/index.ts` (+15) - wires up broker integration
- `README.md` (+53/-1) - docs
- `docs/defaults.md` (+52/-1) - config docs
- `.changeset/smart-cycles-wait.md` (+5) - changeset

**What it does:**
Replaces direct `ctx.ui.custom()` / `ctx.ui.select()` prompts with a configurable broker that routes
approval requests through one or more `ApprovalSource` instances. Supports:

- **Multiple sources**: local Pi UI prompts, remote agent-tick CLI, or custom sources
- **Configurable strategies**: first-terminal wins, all must approve, threshold count, any-approve, veto-threshold
- **Deny policies**: first-deny-veto, all-deny, ignore-denies (with opt-in)
- **Timeout & cancellation**: per-request timeout, operator abort signal, loser cancellation
- **External approval**: agent-tick CLI creates requests visible to remote operators
- **Action fingerprinting**: stable SHA-256 fingerprints for dedup and correlation
- **Grant scope control**: remote sources can be restricted to `once`-only grants

**Key code changes:**

1. **`ApprovalBroker` class** - the core engine:
   - Takes a list of `ApprovalSource` instances + strategy config
   - Each source is started concurrently; decisions are collected and evaluated against the strategy
   - Supports cancellation of losing sources once a terminal decision is reached
   - Validates decisions against request context (brokerRequestId, correlationToken, etc.)
   - Emits detailed event log for debugging/auditing

2. **`ApprovalSource` interface** - abstraction for any approval provider:
   - `start(request, context) → ApprovalHandle`
   - `ApprovalHandle` has `decision: Promise<ApprovalDecision>` and `cancel(reason)`
   - Sources throw `ApprovalSourceStartError` for startup failures

3. **Local source** - wraps existing UI:
   - Uses `ctx.ui.custom()` for rich prompts, falls back to `ctx.ui.select()`
   - Maps `PromptResult` to `ApprovalDecision` with grant scope

4. **Agent-tick source** - external approval via CLI:
   - Spawns `agent-tick request --json-events` subprocess
   - Reads JSON-event stream for `created`, `resolved` events
   - Stores pending requests to a JSON file for reconciliation on restart
   - Supports `abandon` for cancellation

5. **Config integration**:
   - `approvalBroker.enabled` - master switch
   - `approvalBroker.sources` - source definitions (type, bin, timeout, etc.)
   - `approvalBroker.routes.{permissionGate,pathAccess}` - per-feature routing config
   - Default: enabled, single local source, first-terminal strategy

**Our equivalent:**
- Our `extensions/path-access/index.ts` directly uses `ctx.ui.custom()` for prompts
- Our `extensions/guardrails/hooks/permission-gate/index.ts` does the same
- No abstraction for pluggable approval sources exists
- Our `checkAction` / rule-based system evaluates safety synchronously
- We have no external/remote approval mechanism

**Assessment:** This is a large, well-architected feature (~3,500 lines) that significantly changes the
approval flow. The `ApprovalSource` abstraction is clean, and the strategy system is well-designed.
However:
- It's deeply coupled to the old hook architecture - porting would be major surgery
- The agent-tick integration is specific to one external tool
- Our rule-based `checkAction` system would need significant adaptation
- The feature adds substantial complexity for a single-person setup

If we wanted external/remote approval in the future, the broker architecture would be a good reference.
For now, the value-to-effort ratio is low given our current usage pattern.

**Verdict:** ❌ Skip for now - major architectural addition that doesn't align with our current direction.

---

## [yonilerner](https://github.com/k0valik/pi-guardrails/compare/main...yonilerner:pi-guardrails:yoni/path-access-global-grants)

**Branch:** `yoni/path-access-global-grants` · **Status:** diverged · **Commits:** 1 · **Files:** 8

**Overview:** Single focused commit adding a `pathAccess.alwaysScope` config option (`"local"` | `"global"`)
that controls where "Allow … always" grants are persisted. When set to `"global"`, grants are saved to
the user-wide config (`~/.pi/agent/extensions/guardrails.json`) instead of the project-local config.

**⚠️ Architecture mismatch:** Same monolithic structure (`src/hooks/`, `src/config.ts`, `src/commands/`).
Our repo has restructured into `src/core/`, `src/shared/`, `extensions/`, so direct cherry-picking won't work.

---

### Feature: Configurable grant persistence scope

**Commit:** `6ce3fc09`

**Files changed:**

| File | +/- | Change |
|------|-----|--------|
| `src/config.ts` | +21 | Adds `PathAccessAlwaysScope` type, `alwaysScope` field to `PathAccessConfig` & `ResolvedConfig`, and merge logic (memory > local > global priority) |
| `src/hooks/path-access.ts` | +7/-4 | Uses `config.pathAccess.alwaysScope` instead of hardcoded `"local"` for "always" grants |
| `src/hooks/path-access.test.ts` | +261 | Full test suite: `alwaysScope: "local"` saves to local, `"global"` saves to global, validate merge priority |
.| `src/commands/settings-command.ts` | +9 | UI dropdown in settings showing `pathAccess.alwaysScope` with "local" / "global" options |
| `README.md` | +8/-2 | Documents the new config option and scope behavior |
| `docs/defaults.md` | +5 | Defaults docs update |
| `.changeset/` | +10 | Changeset |
| `permission-gate/index.test.ts` | +1/-1 | Trivial test update |

**What it does:**
Adds a `pathAccess.alwaysScope` config field with values `"local"` (default) and `"global"`.

- When `"local"`: "Allow … always" grants are saved to project config (`{project}/.pi/extensions/guardrails.json`)
- When `"global"`: "Allow … always" grants are saved to user-wide config (`~/.pi/agent/extensions/guardrails.json`)
  so the same grant applies in every project
- Merge priority: memory > local > global (granular config scopes win)

**Key code changes:**

1. **New type** `PathAccessAlwaysScope = "local" | "global"` in `src/config.ts`
2. **`alwaysScope` field** added to both `PathAccessConfig` (input) and `ResolvedConfig` (resolved)
3. **Merge logic**: resolves `alwaysScope` from memory > local > global (most specific scope wins)
4. **Hook change**: `const scope = result === "allow-file-session" ? "memory" : alwaysScope;` instead of hardcoded `"local"`
5. **Settings UI**: dropdown in the path-access settings section

**Our equivalent:**
- Our `extensions/path-access/grants.ts` has `persistGrant()` that accepts a `scope: "memory" | "local"`
- Our `extensions/path-access/grants.ts` also has `GrantScope` type - would need `"global"` added
- Our `extensions/path-access/index.ts` determines scope in the prompt result handler
- Config merge lives in `src/shared/config/`

**Assessment:** Clean, well-tested, focused feature. Only ~35 lines of actual logic. The config option
makes the global-scope grant feature (which AlainS7 hardcoded) properly configurable with a sane default.

The same feature ships in AlainS7's fork but hardcoded - this is strictly better.

For our repo, we'd need to:
1. Add `"global"` to our `GrantScope` type in `extensions/path-access/grants.ts`
2. Add `alwaysScope` config option in `src/shared/config/` types + defaults
3. Wire it through `extensions/path-access/index.ts` where `scope` is determined
4. Expose in settings UI

**Verdict:** ⏳ Maybe - useful companion if we ever want global-scope grants. Low effort (~35 lines of logic) to port.
Slightly higher priority if we end up porting the AlainS7 glob compat (they touch related code).

---

## [JJGO](https://github.com/k0valik/pi-guardrails/compare/main...JJGO:pi-guardrails:main)

**Branch:** `main` · **Status:** diverged · **Commits:** 2 · **Files:** 28

**Overview:** Two commits - one package rename (`@mariozechner` → `@earendil-works`, already in upstream),
and one feature commit adding **shared prompt timeouts** with live countdown UI for both path-access and
permission-gate prompts. ~300 lines of actual feature code.

**⚠️ Architecture mismatch** - same monolithic structure (`src/hooks/`, `src/utils/`). Direct cherry-pick won't work.

---

### Feature: Shared prompt timeouts with live countdown

**Commit:** `d6f64c10` (the rename commit `795736a0` is already in `aliou/main`, skip)

**Files changed (feature only):**

| File | +/- | Purpose |
|------|-----|---------|
| `src/utils/prompt-timeout.ts` | +126 | Core module: `createPromptCountdown`, `selectWithOptionalTimeout`, `formatCountdown`, `buildPromptTimeoutReason` |
| `src/hooks/path-access.ts` | +106/-14 | Integrates timeout into path-access custom prompt + fallback select |
| `src/hooks/permission-gate/index.ts` | +72/-9 | Integrates timeout into permission-gate custom prompt + fallback select |
| `src/config.ts` | +11 | Adds `PromptsConfig`, `timeoutSeconds` field, default 300s |
| `src/commands/settings-command.ts` | +60/-2 | Settings UI: "Prompt timeout" submenu, blank = disabled |
| `src/hooks/path-access.test.ts` | +185 | Tests for path-access timeout behavior |
| `src/hooks/permission-gate/index.test.ts` | +105/-3 | Tests for permission-gate timeout behavior |
| `README.md` | +15/-2 | Docs: prompt timeouts section |
| `docs/defaults.md` | +14 | Defaults docs |
.| `package.json` | +15/-19 | Only scope rename |
| `pnpm-lock.yaml` | +793/-865 | Only scope rename |
.| ~11 more files | +1/-1 each | Only import scope renames (`@mariozechner` → `@earendil-works`) |

**What it does:**
Adds a shared `prompts.timeoutSeconds` config option (default: 300s, `null` to disable) that applies to both
path-access and permission-gate prompts. When a timeout is set:

1. **Live countdown** shown in the prompt TUI: `Auto-deny in 4:32`
2. **Auto-deny on expiry**: the operation is blocked with a message that the user appears to be away
3. **RPC/headless fallback**: `selectWithOptionalTimeout` wraps `ctx.ui.select()` with an AbortController

**Key code changes:**

1. **`prompt-timeout.ts`** - the core utility:
   - `createPromptCountdown(timeoutSeconds, tui, onTimeout)`: returns handle with `getSecondsRemaining()` and `dispose()`. Uses `setInterval` at 250ms to request TUI re-renders.
   - `selectWithOptionalTimeout(ui, title, options, timeoutSeconds)`: wraps `ui.select()` with AbortController. Returns `{selection, timedOut}`.
   - `formatCountdown(seconds)`: `4:32` format
   - `buildPromptTimeoutReason(timeout, subject)`: human-readable explanation

2. **Path-access prompt integration**:
   - `createPromptComponent` gets a `timeoutSeconds` parameter
   - Render loop calls `countdown.getSecondsRemaining()` and displays countdown text
   - On timeout, calls `finish("timeout")` which returns new `"timeout"` result
   - `promptForPathAccess` helper: tries `ui.custom()` first, falls back to `selectWithOptionalTimeout`
   - In `setupPathAccessHook`: `"timeout"` result emits blocked event with timeout reason

3. **Permission-gate prompt integration**: same pattern - countdown in custom prompt, fallback select, timeout handling

4. **Config**: `PromptsConfig.timeoutSeconds` - optional number, default 300, `null` = disabled

**Our equivalent:**
- No timeout mechanism exists in our repo
- Our `extensions/path-access/index.ts` uses `ctx.ui.custom()` directly
- Our `extensions/guardrails/hooks/permission-gate/index.ts` uses `ctx.ui.custom()` directly
- The countdown + AbortController logic is independent of hook architecture and could be extracted

**Assessment:** Clean, well-tested, focused feature (~300 lines of logic). The countdown mechanism is
well-designed (250ms interval, proper cleanup with `dispose()`, handles RPC fallback). Adds real UX
improvement - without timeouts, prompts can hang indefinitely if the user walks away.

For our repo, porting would involve:
1. Extract `createPromptCountdown` and `selectWithOptionalTimeout` into a shared utility (e.g., `src/shared/tui/timeout.ts`)
2. Add `prompts.timeoutSeconds` to config types + defaults
3. Wire into `extensions/path-access/prompt.ts` (the TUI component)
4. Wire into `extensions/guardrails/hooks/permission-gate/` prompt

**Verdict:** ✅ Take - high-value UX improvement. ~300 lines of well-tested, self-contained logic.
The core countdown utility is independent of hook architecture and easy to extract.

---

## [kallewoof](https://github.com/k0valik/pi-guardrails/compare/main...kallewoof:pi-guardrails:main)

**Branch:** `main` · **Status:** diverged · **Commits:** 3 · **Files:** 3

**Overview:** Small, focused fork. Only 1 functional commit - the other two are fork documentation
(FORK.md, .gitignore). The functional change adds a `ctx.ui.select()` fallback to path-access when
`ctx.ui.custom()` returns `undefined` (RPC/headless mode), mirroring the same pattern already used
by permission-gate.

**🔧 Architecture match!** This is the first fork we've seen that uses the **same restructured architecture
as our repo** - file at `extensions/path-access/index.ts`. The patch could potentially be cherry-picked
directly, pending minor context differences.

---

### Feature: `ctx.ui.select()` fallback for path-access in RPC mode

**Commits:** `936f494c` (functional) · `e727f5b3` / `949f6edb` (fork docs only)

**Files changed:**

| File | +/- | Change |
|------|-----|--------|
.| `extensions/path-access/index.ts` | +25/-1 | Adds `ctx.ui.select()` fallback when `custom()` returns `undefined` |
| `context/FORK.md` | +86 | Fork documentation (why, what, rebase notes) |
| `.gitignore` | +1 | Adds `package-lock.json` to gitignore |

**What it does:**
When `ctx.ui.custom<PromptResult>(...)` returns `undefined` (which happens in Pi's `--mode rpc`), the
extension now falls back to `ctx.ui.select()` with the same option labels that the TUI prompt renders.
The user's selection is mapped back to the corresponding `PromptResult` value.

**Key code changes:**

1. `const result` → `let result` (needed for reassignment)
2. New fallback block after the `custom()` call:
   - Builds a `[label, result]` array mirroring `FILE_OPTIONS` / `DIR_OPTIONS` from `prompt.ts`
   - Calls `ctx.ui.select()` with the labels
   - Maps selection back via `Array.find()`
   - Defaults to `"deny"` if `select()` also returns `undefined`

This mirrors the exact pattern in `extensions/permission-gate/index.ts` (which already has this fallback).

**Our equivalent:**
- Our `extensions/path-access/index.ts` does NOT have this fallback yet
- Our `extensions/permission-gate/index.ts` already has it (lines 108-116)
- Our file structure matches - the change is a drop-in addition

**Assessment:** Tiny, clean, well-documented fix. Exactly the same pattern as our permission-gate's existing
fallback - just missing from path-access. The FORK.md is excellent documentation.

**Verdict:** ✅ Take - directly applicable. ~25 lines, same architecture, same pattern already used
in our permission-gate. Fixes silent false-denies in RPC mode.

---

## [harrisony](https://github.com/k0valik/pi-guardrails/compare/main...harrisony:pi-guardrails:action-prompted)

**Branch:** `action-prompted` · **Status:** diverged · **Commits:** 1 · **Files:** 5

**Overview:** Single commit adding a `guardrails:action:prompted` event that fires before the user is
prompted for approval. Complements the existing `action:blocked` (post-decision) and `risk:detected`
(pre-UI) events.

**✅ Already in our repo.** Same commit (`0f4f478`), same author, identical changes. Came in through
upstream merges - harrisony's fork is only 8 commits behind upstream, so this landed in upstream/main
before the other forks diverged.

### Assessment
The event is already present in our codebase:
- `GUARDRAILS_ACTION_PROMPTED_EVENT = "guardrails:action:prompted"` in `src/shared/events.ts`
- `emitActionPrompted()` helper function
- Emitted from `extensions/path-access/index.ts` (kind: `confirmation`) before the prompt
- Emitted from `extensions/permission-gate/index.ts` (kind: `permission`) before the prompt

Nothing to do.

**Verdict:** ✅ Already have
