# Fork Audit: Inspect & Evaluate Remote pi-guardrails Forks

## Context (preamble for a new session)

This project is `pi-guardrails` — a personal soft-fork of [aliou/pi-guardrails](https://github.com/aliou/pi-guardrails). The upstream maintainer (aliou) is not actively merging PRs, so other forks have accumulated useful features. The goal is to inspect ~10–15 remote forks, understand what each commit does, and decide if anything is worth cherry-picking or manual porting.

**Key facts about this repo:**
- Package name: `pi-guardrails` (was `@aliou/pi-guardrails`), `private: true`
- Remote: `origin` → `https://github.com/k0valik/pi-guardrails`
- Upstream: `https://github.com/aliou/pi-guardrails`
- **Strictly pnpm only** — never `npm install`, never `npx`
- All tests pass (`pnpm test`)
- Dependencies `@aliou/sh` and `@aliou/pi-utils-settings` are kept as-is
- The upstream `main` has moved ahead (~54 commits behind). Our fork only has the rebranding commit on top.

**Our repo architecture (restructured):**
```
src/
  core/               # Pure guardrail primitives (paths, shell, types)
  shared/             # Pi-extension shared infra (config, events, glob)
extensions/
  path-access/        # Path access feature extension
  permission-gate/    # Permission gate feature extension
  guardrails/         # Core extension (hooks, commands, settings UI)
```

Older forks use the **legacy monolithic architecture**:
```
src/
  hooks/              # path-access.ts, policies.ts, permission-gate/
  utils/              # bash-paths.ts, path.ts, path-access.ts, events.ts
  commands/           # settings-command.ts
```

**Architecture tells you instantly if cherry-picking is possible:**
- **Old architecture** (`src/hooks/`, `src/utils/`): manual port required — files don't map 1:1
- **Same architecture** (`extensions/path-access/`, `extensions/permission-gate/`, `src/shared/`): cherry-pick may apply cleanly

---

## Workflow

### 0. Quick pre-check (recommended first)

Check if the fork's commits are already in our history before doing any deep analysis:

```bash
for sha in <sha1> <sha2> ...; do
  git merge-base --is-ancestor "$sha" origin/main 2>/dev/null \
    && echo "✅ ALREADY HAVE: $sha" \
    || echo "❌ NEW: $sha"
done
```

If all commits are "ALREADY HAVE", skip the fork entirely — just note it in the registry.

### 1. Setup (one-time)

```bash
# Add the upstream as a remote (if not done)
git remote add upstream https://github.com/aliou/pi-guardrails.git
git fetch upstream
```

### 2. Process each compare link

The user will provide links like:
```
https://github.com/aliou/pi-guardrails/compare/main...harrisony:pi-guardrails:action-prompted
```

Extract: `{owner}/{repo}:{branch}` → `owner=harrisony`, `ref=action-prompted`

#### Step A — Get the overview & commit list (one call):

```bash
OWNER="harrisony"
REF="action-prompted"

echo "=== Overview ==="
gh api "repos/aliou/pi-guardrails/compare/main...${OWNER}:pi-guardrails:${REF}" \
  --jq '{ahead_by, behind_by, status, commits: (.commits | length), files: (.files | length)}'

echo "=== Commits ==="
gh api "repos/aliou/pi-guardrails/compare/main...${OWNER}:pi-guardrails:${REF}" \
  --jq '.commits[] | {sha: .sha[0:8], author: .commit.author.name, date: .commit.author.date, message: (.commit.message | split("\n")[0])}'
```

**Key signal: `behind_by`**
- `behind_by >= 45` → fork is based on very old upstream, definitely **old monolithic architecture**
- `behind_by < 15` → fork is recent, likely **same architecture** as our repo
- If file paths in Step B contain `extensions/` or `src/shared/`, it's the new architecture

#### Step B — Get the files changed (architecture + scope check):

```bash
gh api "repos/aliou/pi-guardrails/compare/main...${OWNER}:pi-guardrails:${REF}" \
  --jq '.files[] | {filename, status, additions, deletions}'
```

**Early architecture sniffing:**
| File path pattern | Architecture | Cherry-pick possible? |
|---|---|---|
| `src/hooks/*` | Old monolithic | ❌ Manual port required |
| `src/utils/*` | Old monolithic | ❌ Manual port required |
| `extensions/*/index.ts` | Same as ours | ✅ May apply cleanly |
| `src/shared/*` | Same as ours | ✅ May apply cleanly |
| `src/approval/*` | Old (new dir in old repo) | ❌ Manual port |

**Import scope sniffing:** If the patch references `@mariozechner/pi-coding-agent`, the fork is pre-migration. Our repo uses `@earendil-works/pi-coding-agent`. Imports must be updated during port.

#### Step C — Get the actual patches (save for analysis):

```bash
gh api "repos/aliou/pi-guardrails/compare/main...${OWNER}:pi-guardrails:${REF}" \
  --jq '.files[] | {filename, status, additions, deletions, patch: .patch}' \
  > /tmp/fork-${OWNER}-${REF}.jsonl
```

Then inspect individual file patches:
```bash
# View a specific file's patch
gh api "repos/aliou/pi-guardrails/compare/main...${OWNER}:pi-guardrails:${REF}" \
  --jq '.files[] | select(.filename == "path/to/file.ts") | .patch'

# Quick-preview all patches
python3 -c "
import json, sys
with open('/tmp/fork-${OWNER}-${REF}.jsonl') as f:
    for line in f:
        line = line.strip()
        if not line: continue
        d = json.loads(line)
        fn = d['filename']; patch = d.get('patch','') or ''
        print(f'=== {fn} (+{d[\"additions\"]}/-{d[\"deletions\"]}) ===')
        if patch:
            lines = patch.split(chr(10))
            print(chr(10).join(lines[:25]))
            if len(lines) > 25: print(f'  ... ({len(lines)-25} more lines)')
        print()
"
```

#### Step D — Check if already in our repo:

```bash
gh api "repos/aliou/pi-guardrails/compare/main...${OWNER}:pi-guardrails:${REF}" \
  --jq '.commits[].sha[0:8]' | while read sha; do
  git merge-base --is-ancestor "$sha" origin/main 2>/dev/null \
    && echo "✅ ALREADY HAVE: $sha" \
    || echo "❌ NEW: $sha"
done
```

### 3. Analysis guidelines (learned from practice)

**Group commits by feature area, not by commit order.** A single feature often spans multiple commits
(feature + tests + tweaks). Analyze them as a unit.

**Always check these 5 things for each feature area:**

1. **Architecture match**: old monolith or new restructured?
2. **Already in our repo?** Check commit SHAs + grep for equivalent code
3. **Import scope**: `@mariozechner/` needs update to `@earendil-works/`
4. **Our equivalent**: What does our current codebase do in this area?
5. **Port effort**: Can it be cherry-picked? Manual port? How many files to touch?

**Reporting template (compact):**

```markdown
## [owner](https://github.com/owner/pi-guardrails)

**Branch:** `ref` · **Ahead:** N · **Behind:** M · **Status:** ahead/diverged
**Architecture:** old/same · **Commits:** N · **Files:** M

**Overview:** 1-2 sentence summary of what the fork adds.

### Feature: Feature Name

**Commits:** `abc1234` · `def5678`

| File | +/- | Change |
|------|-----|--------|
| `path/to/file.ts` | +N/-M | What changed |

**What it does:** Paragraph explaining the feature.

**Key code changes:**
- Point 1
- Point 2

**Our equivalent:** What our repo has now (or lacks).

**Assessment:** Is it useful? Any concerns?

**Verdict:** ✅ Take / ❌ Skip / ⏳ Maybe / ✅ Already have
```

**Quick verdict guide:**
| Signal | Verdict |
|--------|---------|
| Already in our history | ✅ Already have |
| Package rename / dead code | ❌ Skip |
| Old architecture, low-value feature | ❌ Skip |
| Old architecture, high-value feature | ✅ Take (manual port) |
| Same architecture, useful fix | ✅ Take (cherry-pick) |
| Large architectural addition | ❌ Skip for now |
| Useful but not urgent | ⏳ Maybe |

---

### 4. Cherry-picking (when architecture matches)

```bash
# Add the fork as a remote
git remote add <owner> https://github.com/<owner>/pi-guardrails.git
git fetch <owner>

# Cherry-pick specific commits
git cherry-pick <sha>
# or try without auto-commit to review first
git cherry-pick --no-commit <sha>
git diff --cached
git restore --staged .    # undo if you don't want it
git cherry-pick --abort   # abort if conflicts are too messy
```

### 5. Manual porting (when architecture differs)

Copy the logic, not the file. The old monolithic structure had:
- `src/hooks/path-access.ts` → maps to `extensions/path-access/index.ts` + `src/core/paths/access.ts`
- `src/utils/path.ts` → maps to `src/core/paths/path.ts`
- `src/utils/bash-paths.ts` → maps to `src/shared/paths/bash-paths.ts`
- `src/utils/bash-intent.ts` → no equivalent, would be new file
- `src/hooks/policies.ts` → maps to `extensions/guardrails/hooks/policies.ts`
- `src/hooks/permission-gate/index.ts` → maps to `extensions/permission-gate/index.ts`
- `src/config.ts` → maps to `src/shared/config/types.ts` + `src/shared/config/defaults.ts`
- `src/commands/settings-command.ts` → maps to `extensions/guardrails/commands/settings-command.ts`

---

## Inspected Forks Registry

Forks already analyzed. Detailed findings in `@forks/audit-findings.md` (part 1) and `@forks/audit-findings-2.md` (part 2 onwards).

| Fork | Branch | Status | Ahead | Behind | Files | Verdict | Key Feature |
|------|--------|--------|-------|--------|-------|---------|-------------|
| [AlainS7](https://github.com/k0valik/pi-guardrails/compare/main...AlainS7:pi-guardrails:main) | `main` | diverged | 14 | 53 | 13 | ✅ Take 2, ⏳ 2, ❌ 1 | Symlink-aware boundary, extension edit gate, glob compat, read-only bash detection |
| [jeprecated](https://github.com/k0valik/pi-guardrails/compare/main...jeprecated:pi-guardrails:main) | `main` | diverged | 2 | 51 | 22 | ❌ Skip | Approval broker routing (~3,500 lines, too architectural) |
| [yonilerner](https://github.com/k0valik/pi-guardrails/compare/main...yonilerner:pi-guardrails:yoni/path-access-global-grants) | `yoni/path-access-global-grants` | diverged | 1 | 45 | 8 | ⏳ Maybe | Configurable `alwaysScope` for global grant persistence |
| [JJGO](https://github.com/k0valik/pi-guardrails/compare/main...JJGO:pi-guardrails:main) | `main` | diverged | 2 | 54 | 28 | ✅ Take | Shared prompt timeouts with live countdown |
| [kallewoof](https://github.com/k0valik/pi-guardrails/compare/main...kallewoof:pi-guardrails:main) | `main` | diverged | 3 | 10 | 3 | ✅ Take | `ctx.ui.select()` fallback for path-access in RPC mode |
| [harrisony](https://github.com/k0valik/pi-guardrails/compare/main...harrisony:pi-guardrails:action-prompted) | `action-prompted` | diverged | 1 | 8 | 5 | ✅ Already have | `guardrails:action:prompted` event |
| [WeiYiAcc](https://github.com/k0valik/pi-guardrails/compare/main...WeiYiAcc:pi-guardrails:feat/guardrails-defaults-and-examples) | `feat/guardrails-defaults-and-examples` | diverged | 3 | 67 | 13 | ✅ Already have | `applyBuiltinDefaults` bridge, tilde fix, docs - all already in our repo |
| [ryanh-ai](https://github.com/k0valik/pi-guardrails/compare/main...ryanh-ai:pi-guardrails:main) | `main` | diverged | 1 | 88 | 2 | ⏳ Maybe | Persistent permission gate grants (project/global scope) |
| [qw457812](https://github.com/k0valik/pi-guardrails/compare/main...qw457812:pi-guardrails:feat/policies-confirmation) | `feat/policies-confirmation` | diverged | 1 | 53 | 9 | ✅ Take | Optional `requireConfirmation` for policy rules (interactive override) |
| [prullanferragut](https://github.com/k0valik/pi-guardrails/compare/main...prullanferragut:pi-guardrails:fix/force-push-example-regex) | `fix/force-push-example-regex` | diverged | 1 | 4 | 1 | ✅ Take | Regex fix for git push --force example pattern (catches flag at any position) |
| [phulot](https://github.com/k0valik/pi-guardrails/compare/main...phulot:pi-guardrails:main) | `main` | diverged | 4 | 71 | 17 | ✅ Take / ⏳ Maybe | Pre-commit hook bypass detection + home-credentials policy rule |
| [xz-dev](https://github.com/k0valik/pi-guardrails/compare/main...xz-dev:pi-guardrails:feature/ignored-bash-args) | `feature/ignored-bash-args` | diverged | 1 | 4 | 12 | ✅ Take | Configurable ignored bash args to reduce false-positive path prompts |
| [nikitakot](https://github.com/k0valik/pi-guardrails/compare/main...nikitakot:pi-guardrails:main-nikitakot) | `main-nikitakot` | diverged | 5 | 61 | 12 | ✅ Take (all 4) | Path filter + protection overrides, askConfirmation, strictAllowSession, read-only mode |

**✅ Take** = worth porting when we get to it
**⏳ Maybe** = useful but lower priority
**❌ Skip** = not worth the effort
**✅ Already have** = already in our repo (came through upstream merges)

---

## Appendix: Architecture Quick Reference

### Old monolithic (pre-restructuring)
```
src/
  hooks/
    path-access.ts           # The path access hook
    policies.ts              # The policies hook
    permission-gate/
      index.ts               # Permission gate hook
      index.test.ts
    path-access.test.ts
    policies.test.ts
  utils/
    path.ts                  # isWithinBoundary, normalizeForDisplay, etc.
    path-access.ts           # checkPathAccess, isPathAllowed
    bash-paths.ts            # extractBashPathCandidates
    bash-intent.ts           # isClearlyReadOnlyBashCommand (if exists)
    events.ts                # emitBlocked, etc.
    prompt-timeout.ts        # createPromptCountdown (if exists)
  approval/                  # (if exists, broker routing)
  config.ts                  # GuardrailsConfig, ResolvedConfig
  commands/
    settings-command.ts      # Settings UI
  index.ts                   # Extension entry
```

### Our restructured architecture
```
src/
  core/
    paths/
      path.ts                # isWithinBoundary, normalizeForDisplay, etc.
      access.ts              # checkPathAccess, isPathAllowed
    shell/
      ast.ts                 # walkCommands
      command-args.ts        # classifyCommandArgs
    types.ts                 # Action, Rule, Safety, etc.
  shared/
    config/
      index.ts               # configLoader
      types.ts               # GuardrailsConfig, ResolvedConfig
      defaults.ts            # DEFAULT_CONFIG
    paths/
      bash-paths.ts          # extractBashPathCandidates
    events.ts                # All public event types + emit helpers
    glob.ts                  # expandGlob
    matching.ts              # Pattern matching utilities
extensions/
  path-access/
    index.ts                 # Path access hook
    prompt.ts                # TUI prompt component
    grants.ts                # Grant persistence
    rules.ts                 # checkAction rule
    targets.ts               # Tool target resolution
  permission-gate/
    index.ts                 # Permission gate hook
    prompt.ts                # TUI prompt component
    grants.ts                # Session grant persistence
    rules.ts                 # checkAction rule
  guardrails/
    hooks/
      policies.ts            # Policy enforcement hook
    commands/
      settings-command.ts    # Settings UI
```
