# Fork Audit Findings (Part 2)

Detailed analysis of remote pi-guardrails forks. Each entry describes what the fork adds, how it
maps to our repo architecture, and a verdict on whether to port.

---

## [xz-dev](https://github.com/k0valik/pi-guardrails/compare/main...xz-dev:pi-guardrails:feature/ignored-bash-args)

**Branch:** `feature/ignored-bash-args` · **Ahead:** 1 · **Behind:** 4 · **Status:** diverged
**Architecture:** same · **Commits:** 1 · **Files:** 12

**Overview:** Adds a configurable `ignoredBashArgs` feature that prevents false-positive path-access
prompts by skipping non-path arguments (URLs, package names, branch names, etc.) during bash
argument extraction.

### Feature: Ignored Bash Args for Path Access

**Commit:** `de36c161` — feat: support ignored bash args for path access

| File | +/- | Change |
|------|-----|--------|
| `src/shared/config/types.ts` | +15/‑0 | `IgnoredBashArgRule` interface + `ignoredBashArgs` on `PathAccessConfig` & `ResolvedConfig` |
| `src/shared/config/defaults.ts` | +1/‑0 | Default empty array |
| `src/shared/config/loader.ts` | +22/‑1 | Merge rules across scopes with dedup |
| `src/shared/config/index.ts` | +1/‑0 | Re-export `IgnoredBashArgRule` |
| `src/shared/paths/bash-paths.ts` | +44/‑2 | `shouldIgnoreBashArg()` + thread `ignoredArgs` through `extractBashPathCandidates()` |
| `src/shared/paths/bash-paths.test.ts` | +51/‑0 | Tests for the filtering logic |
| `extensions/path-access/targets.ts` | +7/‑1 | Thread `ignoredBashArgs` from config through `targetsForTool()` |
| `extensions/path-access/targets.test.ts` | +29/‑0 | Updated tests |
| `extensions/path-access/index.ts` | +8/‑1 | Pass `config.pathAccess.ignoredBashArgs` to `targetsForTool()` |
| `README.md` | +2/‑0 | Documentation |
| `schema.json` | +39/‑0 | JSON Schema update |
| `.changeset/` | +5/‑0 | Changeset |

**What it does:**

`extractBashPathCandidates()` is the function that parses a bash command and extracts path-like
arguments for path-access boundary checking. Currently it treats every non-flag positional argument
that looks like a path as a candidate, which causes false positives for:

- `git clone https://github.com/org/repo.git` — URL looks like a path
- `git branch fix/feature-branch` — "fix/feature-branch" looks like a path
- `pip install requests` — "requests" looks like a path
- `npm install lodash` — same
- `cargo add serde` — same
- `cargo install cargo-edit` — same
- `docker run alpine:latest` — false positive on image references
- `brew install postgresql` — same

This patch introduces `IgnoredBashArgRule`s — per-command rules that tell the system to skip
specific argument tokens during path extraction.

**Key code changes:**

1. **`IgnoredBashArgRule` interface** (types.ts: +15):
   ```ts
   interface IgnoredBashArgRule {
     command: string;          // e.g. "git", "pip", "cargo"
     subcommands?: string[];   // optional prefix match: ["clone"], ["install"]
     argPattern: string;       // substring or regex pattern against the token
     description?: string;
     regex?: boolean;           // when true, argPattern is a regex
   }
   ```

2. **`shouldIgnoreBashArg()` helper** (bash-paths.ts: +38):
   - `basenameOfCommand()` — normalizes command path to basename
   - Matches command name first, then optional subcommand prefix (positional argv comparison),
     then the individual token against the pattern (substring or regex)

3. **Integration** — The list of rules flows from config → `targetsForTool()` → `extractBashPathCandidates()` → filtering happens after `classifyCommandArgs()` extracts each arg:

   ```
   classifyCommandArgs(cmdName, args) → [{token, forcePath}, ...]
         │
         ▼  for each arg:
   shouldIgnoreBashArg(cmdName, args, arg.token, rules)?
         │
         ├─ yes → skip (don't add as path candidate)
         └─ no  → addCandidate(arg.token, arg.forcePath)
   ```

4. **Config merging** (loader.ts) — Rules are deduplicated across global + local + memory scopes
   using `JSON.stringify(normalized)` as a unique key (practical, though not elegant).

**Our equivalent:**

Nothing. Our `extractBashPathCandidates(command, cwd)` takes no filtering rules. Every
`classifyCommandArgs()` output that passes `maybePathLike()` ends up as a path candidate.
`targetsForTool(toolName, input, cwd)` also takes no filtering.

**Assessment:**

- **High-value feature** — false-positive path prompts are a common annoyance that undermines
  trust in guardrails. This is the single biggest UX improvement across all forks analyzed so far.
- **Code quality is excellent** — clean, well-tested (51 lines of tests), same architecture.
- **Zero architecture mismatch** — all file paths map 1:1 to our repo.
- **No import scope issues** — uses `@aliou/sh` (same as ours) and internal relative imports.
- **The config merging** uses `JSON.stringify` for dedup — slightly hacky but works fine for
  config objects. We could tighten it if desired.
- **Schema / README updates included** — changeset file too, following best practices.

**Potential improvements to consider during port:**
- Consider extending the `subcommands` check to handle subcommand aliases (e.g. `pip install` =
  `pip3 install`, `npm i` = `npm install`). The current implementation does strict positional
  comparison (`args[i] !== rule.subcommands[i]`), so `npm i lodash` with a rule for `["install"]`
  won't match. Could normalize common aliases.

**Verdict:** ✅ Take — same architecture, clean code, directly applicable, high UX value.

---

---

## [WeiYiAcc](https://github.com/k0valik/pi-guardrails/compare/main...WeiYiAcc:pi-guardrails:feat/guardrails-defaults-and-examples)

**Branch:** `feat/guardrails-defaults-and-examples` · **Ahead:** 3 · **Behind:** 67 · **Status:** diverged
**Architecture:** old monolith · **Commits:** 3 · **Files:** 13

**Overview:** Three commits by the upstream maintainer (Aliou Diallo) adding `applyBuiltinDefaults`
bridge migration, tilde expansion fix in policy path normalization, documentation for defaults and
examples, and extra example presets. All functional changes are already in our repo - this is an
unmerged feature branch from before the restructuring.

### Feature: applyBuiltinDefaults bridge + bootstrap

**Commits:** `17f54887` - feat: add defaults bridge migration and bootstrap config
`2fc019a5` - feat: add home directory defaults, tilde expansion fix, and command examples

| File | +/- | Change |
|------|-----|--------|
| `src/config.ts` | +60/-2 | `applyBuiltinDefaults` flag on `GuardrailsConfig`+`ResolvedConfig`, migration, bootstrap
| `src/index.ts` | +37/-1 | First-run bootstrap: writes global config if none exists
| `src/utils/migration.ts` | +31/-1 | `migrateApplyBuiltinDefaults` + `needsApplyBuiltinDefaultsMigration`
| `src/hooks/policies.ts` | +20/-4 | Tilde expansion fix in `normalizeTargetForPolicy()`, `fileExists()` param order change
| `src/utils/path.ts` | +18/-0 | New `expandHomePath()` utility
| `src/commands/settings-command.ts` | +112/-0 | Extra example presets (dbt, aws s3, aws iam, aws ec2)
| `package.json` | +2/-1 | Bump `@aliou/pi-utils-settings` from `^0.10.1` to `^0.10.2`

**What it does:**

1. **applyBuiltinDefaults flag** - A new config flag that lets users explicitly opt in or out of
   built-in default policy rules. First-run bootstrap creates a global config with this set to
   `false`. An automatic migration sets it to `true` for existing users (preserves old behavior).
2. **Tilde expansion fix** - `normalizeTargetForPolicy()` and `fileExists()` in the policies hook
   now properly handle `~`-prefixed paths by expanding through `expandHomePath()`.
3. **Documentation** - `docs/defaults.md` mirrors the default config; `docs/examples.md` mirrors
   the example presets from settings UI.
4. **Extra example presets** - dbt run, dbt seed, aws s3 rm, aws iam, aws ec2 terminate added to
   `COMMAND_EXAMPLES` in the settings command.

**All already in our repo:**

| Feature | Our location | Status |
|---------|-------------|--------|
| `applyBuiltinDefaults` flag | `src/shared/config/types.ts` (+ types), `src/shared/config/defaults.ts` (+ default), migration 006 | ✅ Already have |
| `expandHomePath()` | `src/core/paths/path.ts` | ✅ Already have |
| Policies tilde fix | `extensions/guardrails/rules.ts` (`normalizeTarget()` at line 62, `fileExists()` at line 84) | ✅ Already have |
| Extra example presets (dbt, aws) | `extensions/guardrails/commands/settings/examples.ts` (lines 236-276) | ✅ Already have |
| `@aliou/pi-utils-settings` dep | `^0.15.1` in our `package.json` - well past `^0.10.2` | ✅ Already have |

**Not in our repo (and not needed):**
- `docs/defaults.md` - static mirror of the config. Low value; would drift.
- `docs/examples.md` - static mirror of example presets. Low value; would drift.

**Assessment:** This is an unmerged upstream feature branch from before the restructuring.
Everything functional has been ported to the new architecture already. The `behind_by: 67`
confirms this is very old code (oldest fork we've seen).

**Verdict:** ✅ Already have - everything functional is already in our repo.
---

## [ryanh-ai](https://github.com/k0valik/pi-guardrails/compare/main...ryanh-ai:pi-guardrails:main)

**Branch:** `main` · **Ahead:** 1 · **Behind:** 88 · **Status:** diverged
**Architecture:** old monolith · **Commits:** 1 · **Files:** 2

**Overview:** Adds project (`p`) and global (`g`) persistence options to the permission gate
TUI prompt, allowing users to permanently allow a dangerous command by saving it to the local
or global config scope.

### Feature: Persistent permission gate grants

**Commit:** `02c878aa` - feat: add project/global persist options to permission gate dialog

| File | +/- | Change |
|------|-----|--------|
| `src/hooks/permission-gate.ts` | +145/-17 | Add `"allow-project"` + `"allow-global"` to ConfirmResult, sub-prompt for exact/class/cancel, save to configLoader scopes |
| `package-lock.json` | +5205/-0 | npm lockfile (ignore) |

**What it does:**

Extends the permission gate prompt with two new persistence keys alongside the existing
[y/enter: allow, a: allow-session, n/esc: deny]:

- `p` → **allow-project**: saves the allowed pattern to the **local** config scope
- `g` → **allow-global**: saves the allowed pattern to the **global** config scope

After pressing `p` or `g`, a **secondary sub-prompt** asks:
- `e` → **exact command**: saves the raw command string as an `allowedPatterns` entry
- `c` → **command class**: saves the matched pattern as a regex `allowedPatterns` entry
- `esc` → **cancel**: command allowed once, nothing saved (with notification)

The pattern is saved via `configLoader.save()` to the chosen scope (local/global), appending
to existing `allowedPatterns`. The local cache is updated immediately via
`allowedPatterns.push(...compileCommandPatterns([patternToSave]))`.

Also restructures the logic: the `deny` handler is moved from the bottom to immediately after
the prompt result, fixing an ordering bug where deny logic ran last even for allow results.

**Key code changes (permission-gate.ts):**
1. **Type expansion**: `ConfirmResult` gains `"allow-project"` | `"allow-global"`
2. **Help text**: Adds new line `"p: allow for project • g: allow globally"` to the TUI
3. **Keybindings**: `p`/`P` → `"allow-project"`, `g`/`G` → `"allow-global"`
4. **Sub-prompt**: Full inline `ctx.ui.custom()` with a 3-option pattern chooser
   (exact command, command class, cancel). Includes command truncation (60 chars) for display.
5. **Config save**: Reads existing scope config, appends to `allowedPatterns`, saves back.
   Uses `configLoader.save(scope, {...})` for persistence.
6. **Deny ordering**: Moves `deny` handler to immediately after prompt result (was at end of
   function, after session grant logic - a latent ordering bug)

**Our equivalent:**

Our permission gate (`extensions/permission-gate/index.ts`) currently supports:
- `y`/enter → **allow** (one-time, no persistence)
- `a` → **allow-session** (memory scope via `saveCommandSessionGrant()`)
- `n`/esc → **deny**

We do NOT have project/global persistence for permission-gate grants. Our `grants.ts` only
has `saveCommandSessionGrant()` which saves to the memory scope.

**Port complexity:** Moderate.
- ~145 lines to port, spread across `index.ts` (types + routing), `prompt.ts` (keybindings + help text),
  `grants.ts` (new `saveCommandScopeGrant()` function).
- The sub-prompt would need its own component or be inlined as a second `ctx.ui.custom()` call.
- No architecture mismatch beyond the file restructure - the concepts map directly.

**Assessment:**

- **Useful feature** - lets users permanently whitelist commands they trust, reducing friction over time.
  Similar to yonilerner's `alwaysScope` for path-access; these could be bundled into a single
  "persistent grant scopes" feature.
- **Sub-prompt complexity** - the "exact vs class" distinction is good for power users but may confuse
  beginners. The command class (regex) option is smart but depends on having a clean pattern name.
- **Risk** - users could accidentally whitelist a dangerous command globally with one keypress chain.
  The sub-prompt adds a guard but `c` (command class) is broad.
- **Lower priority** than xz-dev (ignored bash args) which reduces false positives for everyone.
**Verdict:** ⏳ Maybe - useful but lower priority; wait until we implement persistent grant scopes
---

## [qw457812](https://github.com/k0valik/pi-guardrails/compare/main...qw457812:pi-guardrails:feat/policies-confirmation)

**Branch:** `feat/policies-confirmation` · **Ahead:** 1 · **Behind:** 53 · **Status:** diverged
**Architecture:** old monolith · **Commits:** 1 · **Files:** 9

**Overview:** Adds an optional `requireConfirmation` flag to `PolicyRule`. When enabled, instead of
silently blocking access to a protected file, the system shows a confirmation dialog explaining which
tool, file, rule, and protection level matched, and asks the user whether to allow the action. If
confirmed, the action proceeds.

### Feature: Optional confirmation for policy-protected files

**Commit:** `1d73d921` - feat(policies): add optional confirmation for protected files

| File | +/- | Change |
|------|-----|--------|
| `src/hooks/policies.ts` | +101/-7 | Core: `requireConfirmation` on CompiledRule/EffectivePolicy, `confirmPolicyAccess()` via `ctx.ui.confirm()`, `getEffectiveProtection()` prefers stricter rules |
| `src/hooks/policies.test.ts` | +202/-0 | Full test suite for confirmation flow
| `src/config.ts` | +2/-0 | `requireConfirmation?: boolean` on `PolicyRule`
| `src/commands/settings-command.ts` | +23/-2 | Settings UI toggle for the new field
| `src/commands/onboarding.ts` | +1/-1 | Minor adjustment
| `docs/defaults.md` | +12/-12 | Documentation mirror
| `README.md` | +5/-1 | Mention in readme
| `.changeset/` | +5/-0 | Changeset

**What it does:**

Normally, when a policy rule matches a file, guardrails blocks the action immediately with only a
notification. This feature adds a per-rule `requireConfirmation` flag that, when `true`, changes the
behavior from silent-block to interactive-prompt:

```
Policy matches file → requireConfirmation?
    │
    ├─ no  → block immediately (current behavior)
    │
    └─ yes → ctx.ui.confirm() with details
               │
               ├─ confirmed → continue (action proceeds)
               │
               └─ denied → block with user-denied reason + emitActionBlocked
```

**Key code changes:**

1. **Config** (`config.ts`): `requireConfirmation?: boolean` added to `PolicyRule`
2. **CompiledRule** (`policies.ts`): gains `requireConfirmation: boolean` (default `false`)
3. **EffectivePolicy** - new interface with `requireConfirmation`, used as return type from
   `getEffectiveProtection()`
4. **`getEffectiveProtection()` ranking** - when two rules have the same protection rank, the one
   WITHOUT `requireConfirmation` (stricter) wins. This means a hard-block rule takes precedence
   over a confirmation rule at the same protection level.
5. **`confirmPolicyAccess()`** - uses `ctx.ui.confirm()` with a structured message:
   - Tool name, file path, rule ID, protection level
   - Policy note (the block message)
   - "Allow this action?" yes/no prompt
6. **`buildPolicyConfirmationMessage()`** - builds a multi-line message string from the effective
   policy details for display in the confirm dialog.
7. **No-UI fallback** - if `!ctx.hasUI`, defaults to blocking (conservative).
8. **Event emission** - when user denies, emits `emitBlocked` with `userDenied: true`

**Our equivalent:**

Our policy system (`extensions/guardrails/rules.ts` + `extensions/guardrails/index.ts`):

- `PolicyRule` (`src/shared/config/types.ts`) - does NOT have `requireConfirmation`
- `CompiledPolicy` (`extensions/guardrails/rules.ts`) - does NOT have `requireConfirmation`
- `setupPolicyHook()` (`extensions/guardrails/index.ts`) - blocks immediately on match, no
  confirmation path
- Our `PolicyMeta` type is `{ ruleId, protection, path }` - no `requireConfirmation` field

The concept maps cleanly to our architecture:
- Config change: add `requireConfirmation?: boolean` to `PolicyRule` in types.ts
- Rule change: add `requireConfirmation` to `CompiledPolicy` + `compilePolicies()`
- Hook change: in `setupPolicyHook()`, after `checkAction()` returns match, look up the matching
  compiled policy's `requireConfirmation` flag. If true, call `ctx.ui.confirm()` before blocking.

**Assessment:**

- **High-value feature** - addresses a real pain point: users currently have to manually edit config
  to allow access to a protected file. This provides a safe, interactive override inline.
- **Well-designed** - clean interface, good prioritization (stricter rules win), no-UI fallback,
  proper event emission.
- **Well-tested** - 202 lines of tests.
- **Code is clean** - the `EffectivePolicy` interface and `getEffectiveProtection()` refactor are
  good abstractions.
- **Port is moderate** - ~3 files in our repo (types.ts, rules.ts, index.ts), no architecture
  mismatch beyond the file restructure.
**Verdict:** ✅ Take - user requested, well-designed, moderate port, high value
---

## [prullanferragut](https://github.com/k0valik/pi-guardrails/compare/main...prullanferragut:pi-guardrails:fix/force-push-example-regex)

**Branch:** `fix/force-push-example-regex` · **Ahead:** 1 · **Behind:** 4 · **Status:** diverged
**Architecture:** same · **Commits:** 1 · **Files:** 1

**Overview:** Fixes the `git push --force` example pattern in the settings UI to use a regex that
catches force-push flags at any position in the command.

### Fix: Git force-push example pattern

**Commit:** `0d86a261` - fix(examples): use regex for git force-push to catch flag at any position

| File | +/- | Change |
|------|-----|--------|
| `extensions/guardrails/commands/settings/examples.ts` | +8/-2 | Replace substring pattern with regex pattern for `git push --force` |

**What it does:**

The old pattern was a simple substring match: `{ pattern: "git push --force" }`. This only matches
when `git push --force` appears as a contiguous substring - it misses:
- `git push origin main --force` (flag at end)
- `git push -f origin main` (short flag)
- `git push --force-with-lease` (variant)

The new pattern uses a regex:
```
git push .*(-f\b|--force(?!-with-lease)|--force-with-lease)
```
This catches `-f`, `--force`, and `--force-with-lease` regardless of position in the git push
arguments. The negative lookahead `(?!-with-lease)` prevents double-matching `--force-with-lease`
(it's explicitly included via the third alternative).

**Same architecture - directly applicable.** The file path matches ours exactly.

**Verdict:** ✅ Take - tiny, clean, directly applicable fix, same architecture

---

## [phulot](https://github.com/k0valik/pi-guardrails/compare/main...phulot:pi-guardrails:main)

**Branch:** `main` · **Ahead:** 4 · **Behind:** 71 · **Status:** diverged
**Architecture:** old monolith · **Commits:** 4 · **Files:** 17

**Overview:** Two unrelated features: (1) a comprehensive `home-credentials` built-in policy rule
covering 16 sensitive home directory file patterns, and (2) pre-commit hook bypass detection that
flags `--no-verify`/`-n` flags and hook-disabling env vars as dangerous patterns in the permission
gate. Also includes 10 test files.

### Feature 1: Pre-commit hook bypass detection ("goes well" with qw457812)

**Commits:** `e2962d64` - feat: bloquer les bypass de hooks pre-commit (--no-verify, HUSKY=0, SKIP=...)
`de3779c7` - test: ajoute des tests pour builtin-matchers, shell-utils et config-defaults

| File | +/- | Change |
|------|-----|--------|
| `src/hooks/permission-gate.ts` | +63/-4 | `checkPreCommitBypass()` function + integration into `findDangerousMatch()` |
| `src/__tests__/pre-commit-bypass.test.ts` | +126/-0 | Tests |
| `src/__tests__/builtin-matchers.test.ts` | +195/-0 | Tests for existing built-in matchers |

**What it does:**

The `checkPreCommitBypass()` function detects two forms of git hook bypass:

1. **Flag-based bypass:** `git commit --no-verify` / `git push --no-verify` (or `-n` short flag)
   - The `-n` detection is clever: checks if the short-flag string contains `n` (e.g. `-nm`, `-an`)
   - Only applies to `git commit` and `git push` subcommands

2. **Env var bypass:** assignments before a git command that disable hooks:
   - `HUSKY=0 git commit` → catches `HUSKY`
   - `HUSKY_SKIP_HOOKS=1 git push`
   - `SKIP=pre-commit git commit`
   - `PRE_COMMIT_ALLOW_NO_CONFIG=true git commit`
   - `GIT_HOOKS_DISABLED=true git push`

It integrates into `findDangerousMatch()` alongside `checkBuiltinDangerous()`, so bypass attempts
are flagged as `(structural)` dangerous patterns and trigger the permission gate's confirmation
prompt.

**Why it pairs well with qw457812:**

Together, these two features create a complete workflow:
- qw457812 adds `requireConfirmation` to policy rules (e.g., a rule protecting `.env` files can be
  set to prompt before blocking rather than silently blocking)
- phulot catches attempts to bypass git hooks when committing/pushing changes to those same
  protected files

Combined: users get a confirmation prompt when touching protected files, AND git hook bypass
attempts are flagged - preventing the agent from "cheating" its way past protections.

**Our equivalent:**
Nothing. Our permission gate has no hook-bypass detection. Our baseline/permission-gate.ts
built-in matchers cover dangerous commands but not `--no-verify` or env-var bypasses.

**Port complexity:** Low-moderate.
- Core logic (`checkPreCommitBypass()`) is a pure function - ~40 lines, no dependencies outside
  `@aliou/sh` (which we use).
- Integration point: bolt into our permission gate's rule system (`extensions/permission-gate/rules.ts`)
  rather than the old monolithic `findDangerousMatch()`.
- The `-n` short-flag detection needs care: `-n` is also `git push -n` (dry run). The phulot code
  checks for `n` anywhere in the short flags, which would flag dry-run pushes - a false positive.
  Worth refining during port.

### Feature 2: `home-credentials` built-in policy rule

**Commit:** `cd0e0e54` - feat: block access to sensitive home directory credentials
`4a9cc357` - test: add unit tests for matching, migration, timing, executor, glob-expander, events, warnings

| File | +/- | Change |
|------|-----|--------|
| `src/config.ts` | +45/-0 | New `home-credentials` rule in DEFAULT_CONFIG with 16 regex patterns |
| 10 test files | +1077/-0 | Tests for existing utilities (matching, migration, etc.) |
| `package.json` / lockfiles | +5897/-0 | Dependency updates (noise) |

**What it does:**

Adds a comprehensive `home-credentials` default policy rule covering 16 credential file patterns:

| Pattern | Tool/Service |
|---------|-------------|
| `/.dbt/profiles.yml` | dBT |
| `/.aws/credentials`, `/.aws/config` | AWS |
| `/.ssh/` | SSH keys and config |
| `/.kube/config` | Kubernetes |
| `/.docker/config.json` | Docker |
| `/\.pgpass$` | PostgreSQL |
| `/\.my\.cnf$` | MySQL |
| `/\.netrc$` | cURL/network credentials |
| `/\.npmrc$` | npm tokens |
| `/\.pypirc$` | PyPI tokens |
| `/.gnupg/` | GnuPG |
| `/.config/gcloud/` | Google Cloud |
| `/.azure/` | Azure |
| `/\.terraformrc$`, `/.terraform.d/credentials.tfrc.json` | Terraform |
| `/\.vault-token$` | HashiCorp Vault |

All patterns use regex matching (`regex: true`), anchored with `^` implicit via substring but
with path segments like `/.aws/` to avoid matching unrelated paths. The rule has
`onlyIfExists: false` and is enabled by default (no `enabled: false`).

**Our equivalent:**

We have similar but more conservative defaults:
- `secret-files` - .env files (enabled)
- `home-ssh` - SSH keys (disabled by default)
- `home-config` - gcloud, gh, op, sops config (disabled by default)
- `home-gpg` - GPG keys (disabled by default)

The phulot rule is more aggressive: covers more tools (dbt, Docker, npm, PyPI, Azure, Terraform,
Vault, .netrc) and defaults to `enabled: true` with `onlyIfExists: false`. Many of our rules are
disabled by default.

**Verdict on the rule itself:** Worth adopting the additional patterns into our existing rules or
adding a new `home-credentials` rule, but with more conservative defaults (disabled by default,
`onlyIfExists: true`) to match our philosophy.
**Overall verdict for phulot:**
- Pre-commit bypass detection: ✅ Take (pairs with qw457812)
- Home-credentials rule: ⏳ Maybe (patterns useful, but adopt selectively)
- Test files: already covered in our codebase
---

## [nikitakot](https://github.com/k0valik/pi-guardrails/compare/main...nikitakot:pi-guardrails:main-nikitakot)

**Branch:** `main-nikitakot` · **Ahead:** 5 · **Behind:** 61 · **Status:** diverged
**Architecture:** old · **Commits:** 5 · **Files:** 12

**Overview:** 5 focused commits adding 4 feature areas: (1) path filters and
allowed-pattern protection overrides for policies, (2) ask-confirmation dialogs
for blocked policy rules, (3) strict permission-gate allow-session matching, and
(4) a full read-only mode with CLI flag, toggle command, and runtime state
management.

Two of these overlap with features we've already rated: `askConfirmation` is
essentially the same as qw457812's `requireConfirmation` (✅ Take), and
`strictAllowSession` is a small additive improvement to permission-gate.
The path-filter and read-only mode are novel.

---

### Feature 1: Path Filter + Allowed-Pattern Protection Overrides

**Commits:** `0ae76da3` (path filter) · `fd760106` (protection override)

| File | +/- | Change |
|------|-----|--------|
| `src/config.ts` | +54/-10 | Splits `PatternConfig` into `FilePatternConfig` (adds `pathFilter`, `protection`) and `CommandPatternConfig` (adds `strict`); extends `PolicyRule.patterns`/`allowedPatterns` types |
| `src/utils/matching.ts` | +52/-12 | Adds `CompiledFilePattern`/`CompiledCommandPattern` generics; `compileFilePattern` accepts a `cwd` parameter for pathFilter resolution; `compileCommandPattern` handles `strict` mode |
| `src/utils/path.ts` | +58/-2 | Adds `resolvePathFilter`, `isPathInside`, `isPathInsideAny` helpers |

**What it does:**
Two tightly related changes:

**1. `pathFilter` on policy patterns** - each `PolicyRule.pattern` entry can
optionally include a `pathFilter: string[]` that restricts where the pattern
applies. Supports relative (`.` `./` `../`), home (`~`), and absolute paths.
A pattern only matches if the file is inside *any* of the filter paths. If
unset, matches everywhere (current behavior).

Example: `"pathFilter": ["."]` restricts to CWD only.

**2. `protection` override on `allowedPatterns`** - exceptions can optionally
specify a `protection` level (`readOnly`, `noAccess`) instead of the default
full-access behavior. This lets you create nuanced rules like:

```json
{
  "patterns": [{"pattern": "^", "regex": true}],
  "allowedPatterns": [
    {"pattern": "^", "regex": true, "pathFilter": ["."]},
    {"pattern": "~/.pi/**", "protection": "readOnly"}
  ],
  "protection": "noAccess"
}
```

This means: block everything outside CWD, except allow read-only to `~/.pi`.

**Key code changes:**
- `FilePatternConfig` extends `PatternConfig` with `pathFilter?: string[]` and `protection?: Protection`
- `CommandPatternConfig` extends `PatternConfig` with `strict?: boolean`
- `compileFilePattern` now accepts `cwd: string` and checks `isPathInsideAny()` when pathFilter is set
- `isPathInsideAny` resolves each filter via `resolvePathFilter` and checks containment
- `compileFilePatterns` now returns `CompiledFilePattern[]` (was `CompiledPattern[]`)
- `getEffectiveProtection` iterates allowed patterns, checking protection overrides

**Our equivalent:**
- `src/shared/config/types.ts` - `PolicyRule.patterns` is `PatternConfig[]`, no pathFilter or per-pattern protection
- `src/shared/matching.ts` - `compileFilePattern` doesn't accept `cwd`; no `CompiledFilePattern`
- `src/core/paths/path.ts` - no `isPathInside`/`isPathInsideAny`/`resolvePathFilter`
- `src/core/paths/access.ts` - `checkPathAccess` uses lexical boundary (`isWithinBoundary`) not path-filter containment

**Assessment:** Well-designed feature that enables precise policy scoping.
The pathFilter concept is more intuitive than our current `onlyIfExists` +
boundary-only approach for restricting policies to CWD. The protection-override
on allowedPatterns is powerful - it turns exceptions from a binary allow/block
into a graded mechanism (allow read-only, allow full, override to more restrictive).
~60 lines of actual logic + matching infra.

**Verdict:** ✅ Take - useful policy UX improvement, complements qw457812's
`requireConfirmation`. The pathFilter concept is reusable across rules.

---

### Feature 2: askConfirmation on Policy Rules

**Commits:** `2d03b663`

| File | +/- | Change |
|------|-----|--------|
| `src/hooks/policies.ts` | +138/-14 | Adds `askConfirmation` to `CompiledRule`; in the hook, after matching a blocking rule, checks `askConfirmation` and shows confirmation dialog; on `allow-session` saves to memory scope |
| `src/config.ts` | +54/-10 | Adds `askConfirmation?: boolean` to `PolicyRule` |
| `src/utils/confirmation-ui.ts` | +282 | Shared TUI component `createConfirmationUI` used by both permission-gate and policies |
| `src/commands/settings-command.ts` | +25/-0 | Adds `askConfirmation` field to the settings UI rule editor |
| `README.md` | +50/-2 | Docs for `askConfirmation` and confirmation dialog |

**What it does:**
Adds an `askConfirmation` boolean to each `PolicyRule`. When `true` and the rule
would block access, instead of silently blocking, the system shows a confirmation
dialog (Allow once / Allow session / Deny). If the user allows, the action
proceeds without blocking. If denied, the block reason includes `userDenied: true`.

The `allow-session` grant persists to the `memory` scope by adding the file path
to the rule's `allowedPatterns` in memory.

**Key code changes:**
- `CompiledRule` gains `askConfirmation: boolean` (threaded from config)
- After `getEffectiveProtection` determines a match, before blocking:
  - If `askConfirmation && ctx.hasUI`: show `createConfirmationUI()` dialog
  - If `allow-session`: save to memory-scope config + update local compiled cache
  - If `deny`: emit blocked event with `userDenied: true`
  - If `allow` or `allow-session`: `continue` (don't block)
- In headless mode (`!ctx.hasUI`): blocks by default with a message

**Our equivalent:**
- Our `extensions/guardrails/rules.ts` has `CompiledPolicy` without `askConfirmation`
- Our `extensions/guardrails/index.ts` blocks immediately on matching - no confirmation path
- qw457812's `requireConfirmation` is the same feature with a different name

**Assessment:** This is essentially qw457812's `requireConfirmation` feature, but
with a shared UI component (`confirmation-ui.ts`) that both permission-gate and
policies use. The `createConfirmationUI` component is cleanly extracted - it's a
generic TUI prompt factory with scroll support, configurable border colors, and
explanation display.

The `allow-session` persistence to memory scope is similar to qw457812's approach.

**Verdict:** ✅ Take - same feature as qw457812 but better extracted (shared UI
component). When porting, we should use nikitakot's `createConfirmationUI`
architecture rather than inlining the prompt in the policies hook.

---

### Feature 3: Strict Permission-Gate Allow-Session Matching

**Commits:** `cb105d43`

| File | +/- | Change |
|------|-----|--------|
| `src/config.ts` | +54/-10 | Adds `strictAllowSession?: boolean` to `PermissionGateConfig`; `CommandPatternConfig` with `strict` field |
| `src/hooks/permission-gate.ts` | +53/-264 | On `allow-session`, uses `strict: true` pattern when `strictAllowSession` is enabled |
| `src/utils/matching.ts` | +52/-12 | `compileCommandPattern` handles `strict` → exact trimmed match |
| `README.md` | +50/-2 | Documents `strictAllowSession` config option |

**What it does:**
Adds a `permissionGate.strictAllowSession` config option (default `false`). When
enabled, the "allow for this session" option saves the *exact* command string
with `strict: true` mode, which requires an exact trimmed match instead of the
default substring match.

Without strict: allowing `rm -rf /tmp` also allows `echo rm -rf /tmp` (substring).
With strict: only the exact `rm -rf /tmp` (trimmed) matches.

**Key code changes:**
- `CommandPatternConfig` adds `strict?: boolean`
- `compileCommandPattern`: when `strict && !regex`, test is `input.trim() === config.pattern.trim()`
- In `setupPermissionGateHook`: on `allow-session`, if `strictAllowSession`, creates `{ pattern: command, strict: true }` instead of `{ pattern: command }`
- Warning if both `strict` and `regex` are set (strict is ignored)

**Our equivalent:**
- Our `extensions/permission-gate/index.ts` saves `{ pattern: command }` (always substring)
- Our `src/shared/matching.ts` `compileCommandPattern` doesn't have `strict` mode

**Assessment:** Tiny feature (~10 lines of logic). Addresses a real concern -
accidentally allowing dangerous substrings via session grants. Low effort to port.

**Verdict:** ✅ Take - small, well-defined, directly applicable.

---

### Feature 4: Read-only Mode

**Commits:** `27edf419`

| File | +/- | Change |
|------|-----|--------|
| `src/readonly-state.ts` | +116 | New module: runtime state management, CLI flag, config restoration, status indicator |
| `src/index.ts` | +37/-0 | Registers `--readonly` flag, `/guardrails:readonly` command, initializes state on startup, restores on `session_tree` |
| `src/hooks/permission-gate.ts` | +53/-264 | When readonly active: ALL bash commands treated as dangerous (overrides normal matching) |
| `src/hooks/policies.ts` | +138/-14 | When readonly active: upgrades `none` protection to `readOnly` |
| `src/hooks/index.ts` | +3/-2 | Passes `getReadonlyEnabled` function to both hooks |
| `src/config.ts` | +54/-10 | Adds `readOnlyMode.enabled` to config types + defaults |
| `src/commands/settings-command.ts` | +25/-0 | Settings UI: "Readonly Mode" section with enable/disable toggle |
| `README.md` | +50/-2 | Docs for readonly mode |

**What it does:**
Adds a runtime read-only mode that restricts destructive operations:

- **CLI flag**: `--readonly` starts Pi in read-only mode
- **Toggle command**: `/guardrails:readonly` toggles at runtime
- **Status indicator**: `[readonly]` badge in the status line
- **Permission gate**: ALL bash commands are treated as dangerous (skips normal pattern matching)
- **Policies**: upgrades `none` protection to `readOnly` for all policy rules
- **Persistence**: state saved via `pi.appendEntry` and restored on session reload/tree navigation
- **Config support**: `readOnlyMode.enabled` in config (merges global/local/memory)

**Key code changes:**
1. **`readonly-state.ts`**:
   - `currentReadonlyEnabled` - in-memory state variable
   - `getReadonlyEnabled()` - getter for hooks
   - `setReadonlyState(enabled, pi, ctx?)` - setter + persist + status
   - `initializeReadonlyState(pi, ctx)` - checks `--readonly` flag first, then config
   - `restoreReadonlyState(ctx, pi)` - on reload, checks session branch for persisted state
   - `updateStatus(ctx)` - shows/hides `[readonly]` badge
2. **Permission gate**: if `getReadonlyEnabled()`, skips `findDangerousMatch()` and creates a synthetic match: `{ description: "bash command in readonly mode", pattern: "(readonly-mode)" }`
3. **Policies**: in `getEffectiveProtection`, if readonly and finalProtection is `"none"`, upgrades to `readOnly`

**Our equivalent:**
- Nothing equivalent exists in our repo
- No runtime mode, no CLI flag, no toggle command
- Our hooks don't accept a `getReadonlyEnabled` function

**Assessment:** Well-designed feature (~120 lines of logic + 80 lines of config/entry wiring).
The architecture is clean: hooks accept a `getReadonlyEnabled` function pointer
rather than being coupled to a global, making it testable. State persistence
via `pi.appendEntry` is the standard Pi pattern.

The implementation is conservative: readonly mode doesn't block reads, it just
adds friction (upgrades `none` to `readOnly`, flags all bash as dangerous).
This matches the user goal of "prevent accidents, not exploration".

**Verdict:** ✅ Take - well-designed, self-contained, high-value safety feature.

---

### Summary

| Feature | Verdict | Effort | Notes |
|---|---|---|---|
| Path filter + protection overrides | ✅ Take | Medium | ~60 lines logic + matching infra + config types, splits PatternConfig into file/command variants |
| askConfirmation (policies) | ✅ Take | Medium | Same as qw457812's `requireConfirmation` - use nikitakot's `createConfirmationUI` architecture |
| strictAllowSession | ✅ Take | Low | ~10 lines of logic, config option, prevents substring-matching accidents |
| Read-only mode | ✅ Take | Medium | ~120 lines logic + 80 lines wiring, CLI flag, toggle command, state persistence |

**Bottom line:** All 4 features are worth taking. The path filter (Feature 1)
and read-only mode (Feature 4) are novel additions to our audit collection.
`askConfirmation` overlaps with qw457812 but provides a better architecture
(shared UI component). `strictAllowSession` is a tiny but useful add-on.

See `@forks/fork-audit.md` for the workflow and registry of all inspected forks.
