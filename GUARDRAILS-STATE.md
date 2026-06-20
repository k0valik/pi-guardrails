# Guardrails — Divergence & Roadmap

Tracked divergences from [`aliou/pi-guardrails`](https://github.com/aliou/pi-guardrails) `main`.

## Upstream Status

- **Upstream:** `aliou/pi-guardrails` — maintainer not actively merging PRs
- **Our fork:** `k0valik/pi-guardrails` — tracks upstream `main` + rebranding + manual ports from forks
- **Architecture:** upstream restructured into `src/core/`, `src/shared/`, `extensions/` — we match this
- **Package:** `pnpm` only, `@earendil-works/pi-coding-agent` scope, `@aliou/pi-utils-settings@^0.15.1`

## Forks Analyzed

12 forks audited. Detailed analysis in `forks/audit-findings.md` and `forks/audit-findings-2.md`.
Workflow in `forks/fork-audit.md`.

### ✅ Take — worth porting

| Fork | Feature | ETA |
|------|---------|-----|
| [xz-dev](https://github.com/k0valik/pi-guardrails/compare/main...xz-dev:pi-guardrails:feature/ignored-bash-args) | Ignored bash args (reduce false-positive path prompts) | Not yet |
| [qw457812](https://github.com/k0valik/pi-guardrails/compare/main...qw457812:pi-guardrails:feat/policies-confirmation) | `requireConfirmation` for policy rules (interactive override) | Not yet |
| [phulot](https://github.com/k0valik/pi-guardrails/compare/main...phulot:pi-guardrails:main) | Pre-commit hook bypass detection (pairs with qw457812) | Not yet |
| [prullanferragut](https://github.com/k0valik/pi-guardrails/compare/main...prullanferragut:pi-guardrails:fix/force-push-example-regex) | Better regex for `git push --force` example | Not yet |
| [JJGO](https://github.com/k0valik/pi-guardrails/compare/main...JJGO:pi-guardrails:main) | Shared prompt timeouts with live countdown | Not yet |
| [kallewoof](https://github.com/k0valik/pi-guardrails/compare/main...kallewoof:pi-guardrails:main) | `ctx.ui.select()` fallback for path-access in RPC mode | Not yet |
| [AlainS7](https://github.com/k0valik/pi-guardrails/compare/main...AlainS7:pi-guardrails:main) | Symlink-aware boundary checks + extension edit gate | Not yet |

### ✅ Already have — came through upstream merges

- [harrisony](https://github.com/k0valik/pi-guardrails/compare/main...harrisony:pi-guardrails:action-prompted) — `guardrails:action:prompted` event
- [WeiYiAcc](https://github.com/k0valik/pi-guardrails/compare/main...WeiYiAcc:pi-guardrails:feat/guardrails-defaults-and-examples) — `applyBuiltinDefaults`, tilde fix, example presets

### ⏳ Maybe — useful but lower priority

- [yonilerner](https://github.com/k0valik/pi-guardrails/compare/main...yonilerner:pi-guardrails:yoni/path-access-global-grants) — configurable `alwaysScope` for global grants
- [ryanh-ai](https://github.com/k0valik/pi-guardrails/compare/main...ryanh-ai:pi-guardrails:main) — persistent permission-gate grants (project/global scope)
- [phulot](https://github.com/k0valik/pi-guardrails/compare/main...phulot:pi-guardrails:main) — home-credentials policy rule patterns
- [AlainS7](https://github.com/k0valik/pi-guardrails/compare/main...AlainS7:pi-guardrails:main) — glob compat, read-only bash detection

### ❌ Skip

- [jeprecated](https://github.com/k0valik/pi-guardrails/compare/main...jeprecated:pi-guardrails:main) — approval broker routing (too large, too architectural)

## Key Architecture Decisions

- **Policy rules live in `extensions/guardrails/rules.ts`** (compiled + checked via `PolicyMeta`)
- **Permission gate in `extensions/permission-gate/`** with `index.ts`, `prompt.ts`, `grants.ts`, `rules.ts`
- **Path access in `extensions/path-access/`** with same structure
- **Config in `src/shared/config/`** with typed scopes (global / local / memory)
- **Migration 006** handles `applyBuiltinDefaults` bridge
- **Events** in `src/shared/events.ts`: `action:blocked`, `action:prompted`, `risk:detected`, `feature:request`, `feature:register`

## Porting Notes

- Most forks use old monolithic arch (`src/hooks/`, `src/utils/`). Ports are manual.
- xz-dev and prullanferragut use same arch — directly applicable.
- qw457812 + phulot pair well: qw457812 adds `requireConfirmation` to policy rules; phulot catches `--no-verify` / `HUSKY=0` bypass attempts via the permission gate.

### Architecture sniffing
- **`behind_by >= 45`** → old monolithic (pre-restructuring). Manual port required.
- **`behind_by < 15`** → same architecture as us. Cherry-pick may apply cleanly.
- **Import scope**: `@mariozechner/pi-*` = pre-migration. Our scope is `@earendil-works/pi-*`.

### Old → new file mapping (manual ports)
| Old monolith | Our restructured |
|---|---|
| `src/hooks/path-access.ts` | `extensions/path-access/index.ts` + `src/core/paths/access.ts` |
| `src/utils/path.ts` | `src/core/paths/path.ts` |
| `src/utils/bash-paths.ts` | `src/shared/paths/bash-paths.ts` |
| `src/utils/bash-intent.ts` | no equivalent (new file) |
| `src/hooks/policies.ts` | `extensions/guardrails/hooks/policies.ts` |
| `src/hooks/permission-gate/index.ts` | `extensions/permission-gate/index.ts` |
| `src/config.ts` | `src/shared/config/types.ts` + `src/shared/config/defaults.ts` |
| `src/commands/settings-command.ts` | `extensions/guardrails/commands/settings-command.ts` |
| `src/utils/events.ts` | `src/shared/events.ts` |

---

## Session Log

### 2026-06-17 - nikitakot audit + upstream lift session

**Forks analyzed (added to audit):**
- [nikitakot](https://github.com/k0valik/pi-guardrails/compare/main...nikitakot:pi-guardrails:main-nikitakot) - 4 features, all ✅ Take: path filter + protection overrides, askConfirmation, strictAllowSession, read-only mode

**Documents created:**
- `forks/feature-comment-denied-reason.md` - design doc for configurable `comment` field on deny feedback (novel feature, not in any fork)
- All audit docs updated to use comparison URLs against `k0valik/pi-guardrails`

**Upstream lifts (from `aliou/main`):**
- `9674ac5` - fix: stamp guardrails config version on save (self-contained loader fix)
- `17ae843` - fix(examples): git force-push regex (prullanferragut's fix, accepted upstream)
- `9c9bd06` - feat(path-access): dynamic Pi docs/skill paths + `/dev/null` default (cherry-picked, no migration)

**DevDep bumps:**
- `@earendil-works/pi-coding-agent` `0.74.0` → `0.79.9`
- `@earendil-works/pi-tui` `0.74.0` → `0.79.9`
