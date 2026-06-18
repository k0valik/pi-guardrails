# Convergence Audit: Feature Pain-Point Clusters Across Forks

> **Status:** Discussion document — not a decision or roadmap
> **Purpose:** Identify where independent forks independently address the same pain points, so we can think in clusters rather than individual features
> **Date:** 2026-06-18

---

## 0. Clarification: The `comment` Feature Has Two Audiences

The `feature-comment-denied-reason.md` design proposal mixed two concerns into one field.
You've clarified the split:

| Channel | What to show | Audience |
|---------|-------------|----------|
| **UI prompt** (for you) | The **trigger** - which rule/pattern matched | Human user |
| **Deny message** (sent back) | The **`comment`** - guidance text like "Use pnpm instead" | Model/LLM |

These are **complementary**, not alternatives. Both should exist:

---

### UI: The Trigger Indicator (for the human user)

When the prompt fires, the top line shows what *caught* the action:

```
╔══════════════════════════════════════╗
║  ⚠ Rule: admin-commands              ║  ← trigger: rule name
║     Pattern: npm install              ║  ← trigger: matched pattern
║  ─────────────────────────────       ║
║  The model wants to run:             ║
║  $ npm install lodash                ║
║                                      ║
║  Allow execution?                    ║
╚══════════════════════════════════════╝
```

This surfaces metadata the system **already has** (`safety.metadata.description`,
`safety.metadata.pattern`, `safety.reason`) - no new config fields needed.
It answers: *"Which guardrail caught this?"* at a glance.

### Model feedback: The `comment` field (for the LLM)

The `comment` field from the original proposal remains. It's a per-pattern/per-rule
string that is:

1. **Not** rendered in the UI prompt (the trigger indicator replaces that)
2. **Sent to the model** when the action is denied, as part of the deny reason:

   ```
   "Blocked by rule: dependency installation. Use pnpm instead"
   ```

The `comment` field still needs to be added to `PatternConfig` / `PolicyRule`,
threaded through the metadata pipeline, and included in the deny message.
The original proposal's sections 3 (Return comment to model) and 5 (Settings UI)
are still valid - just skip section 2 (Display comment in TUI prompt).

### The confusion (my bad)

My earlier draft in Section 0 presented these as a binary choice - trigger *or* comment.
That was wrong. The correct design is:

- **UI**: Show the trigger (dynamic - what matched)
- **Model**: Send the comment (author-defined - what to do instead)

Both live in the same deny flow, just rendered to different consumers.

### Surface area for both

| Concern | What to add | Where | Config change? |
|---------|------------|-------|----------------|
| Trigger in UI | Reorder prompt component to show `safety.metadata` info at top | `extensions/permission-gate/prompt.ts`, `extensions/path-access/prompt.ts` | ❌ No - uses existing metadata |
| Trigger in policy blocks | Surface `PolicyMeta` in the block notification | `extensions/guardrails/index.ts` | ❌ No |
| `comment` field | Add `comment?: string` to `PatternConfig`, `PolicyRule` | `src/shared/config/types.ts` | ✅ Yes - new optional field |
| `comment` → model | Use `safety.metadata.comment` in deny reason on deny | `extensions/permission-gate/index.ts`, `extensions/permission-gate/rules.ts` | ❌ No - reads the new field |
| `comment` → model (policies) | Thread comment through `PolicyMeta` → deny reason | `extensions/guardrails/rules.ts`, `extensions/guardrails/index.ts` | ❌ No |
| `comment` in settings | Add field to pattern editor + rule editor | `extensions/guardrails/components/pattern-editor.ts`, `extensions/guardrails/commands/settings/index.ts` | ❌ No - UI reads config shape |

**Architecture fit overall:** ✅ Clean. The two concerns touch different layers (UI
rendering vs data model) and don't conflict. The `comment` field is ~5 lines in
types.ts; the trigger indicator is ~20 lines of prompt reordering.

---

## 1. The Five Convergence Clusters

Across all 13 forks and ~22 feature areas, I see **five distinct pain-point clusters** where multiple independent authors independently converged on similar solutions. Each cluster represents a genuine unmet need.

---

### Cluster A: "Let Me Override a Block Interactively"

**The pain point:** Guardrails blocks something. To allow it, I have to edit config files by hand, restart the session, or live with the block. There's no inline "no, this is fine, let it through" flow.

**Forks that hit this:**

| Fork | Feature | Mechanism |
|------|---------|-----------|
| qw457812 | `requireConfirmation` on PolicyRule | `ctx.ui.confirm()` dialog with allow/allow-session/deny options |
| nikitakot | `askConfirmation` on PolicyRule | Same concept, but with extracted `createConfirmationUI` shared component |
| ryanh-ai | Persistent permission gate grants | `p`/`g` keys to save allowed patterns to project/global scope |
| nikitakot | path filter + protection overrides | Granular exceptions via `allowedPatterns` with per-entry `protection` overrides |

**Semantic convergence:** All four are responses to "I need to override a block, and editing JSON is not the right workflow." They differ in scope (policy vs permission-gate, session vs persistent) but the core UX pattern is identical: **interactive allow-with-granularity.**

**Architecture surface area if we implement this cluster:**

| What | Files touched | Our fit |
|------|---------------|---------|
| Policy `askConfirmation` flag | `src/shared/config/types.ts` (+1 field on `PolicyRule`), `extensions/guardrails/rules.ts` (+field on `CompiledPolicy`), `extensions/guardrails/index.ts` (+branch in hook loop) | ✅ Clean — minimal new surface |
| Shared confirmation UI component | `extensions/shared/` or `extensions/guardrails/components/confirm-ui.ts` | ⚠️ **Tension:** nikitakot extracted `createConfirmationUI` as a shared component used by both permission-gate AND policies. Our architecture puts them in separate extension dirs. A shared component would need to live somewhere both can import — either a new `extensions/shared/` or in `src/shared/tui/`. This is a new directory. |
| Persistent grant (project/global) for permission gate | `extensions/permission-gate/grants.ts` (new `saveCommandScopeGrant`), `extensions/permission-gate/index.ts` (new keybindings), `extensions/permission-gate/prompt.ts` (help text + sub-prompt) | ✅ Clean — fits existing extension structure |
| Path filter on policy patterns | `src/shared/config/types.ts` (new `FilePatternConfig`), `src/shared/matching.ts` (cwd-aware compile), `src/core/paths/path.ts` (new `isPathInside` helpers) | ✅ Clean — but would benefit from **not** splitting `PatternConfig` (see notes below) |

**⚠️ Watch item — PatternConfig split:** Both nikitakot's path filter feature and the `comment` feature touch `PatternConfig`. If we add fields incrementally (`comment?`, `strict?`, `pathFilter?`, `protection?`) to the existing `PatternConfig`, it stays simple. If we follow nikitakot's approach of splitting into `FilePatternConfig` + `CommandPatternConfig`, that's a larger refactor. Recommend: **resist the split** unless we're porting the full path-filter feature. Extra optional fields are fine.

---

### Cluster B: "Stop Bothering Me with Noise"

**The pain point:** Too many false-positive prompts. Guardrails flags things that are obviously safe (URLs as paths, read-only commands, package names as files). Users start reflex-dismissing prompts, which undermines the whole system.

**Forks that hit this:**

| Fork | Feature | Mechanism |
|------|---------|-----------|
| xz-dev | `ignoredBashArgs` | Per-command rules to skip non-path tokens during bash arg extraction |
| AlainS7 | Read-only bash detection | `isClearlyReadOnlyBashCommand` — skip policy enforcement for `ls`, `cat`, `rg`, etc. |
| JJGO | Prompt timeouts | Auto-deny prompts after N seconds of inactivity (reduces stale prompts) |
| nikitakot | Read-only mode | Master switch: upgrades all protections, flags all bash as dangerous |

**Semantic convergence:** All four reduce friction, but from different angles:
- **xz-dev** reduces false-positive **path candidates** (pre-extraction filtering)
- **AlainS7** reduces false-positive **policy blocks** (post-match bypass for read-only ops)
- **JJGO** prevents **stale prompts** from hanging indefinitely
- **nikitakot** provides a **global safety switch** (coarser, more opinionated)

The first three are complementary and could coexist. nikitakot's read-only mode is a different class of feature (it *increases* friction intentionally).

**Architecture surface area:**

| What | Files touched | Our fit |
|------|---------------|---------|
| `ignoredBashArgs` rules | `src/shared/config/types.ts` (+`IgnoredBashArgRule`), `src/shared/paths/bash-paths.ts` (+filtering logic), `extensions/path-access/targets.ts` (+threading) | ✅ **Best fit in the whole audit** — same architecture, 1:1 file mapping, no refactoring needed |
| Read-only bash detection | `src/core/shell/command-args.ts` or new `src/shared/shell/bash-intent.ts` + `extensions/guardrails/hooks/policies.ts` (bypass) | ✅ Clean — pure function, no config changes needed if we use a built-in allowlist |
| Prompt timeouts | `src/shared/tui/timeout.ts` (new utility), `extensions/permission-gate/prompt.ts` + `extensions/path-access/prompt.ts` (wire in) | ✅ Clean — `src/shared/tui/` is a natural addition; no config changes needed beyond a `prompts.timeoutSeconds` field |
| Read-only mode | New `extensions/guardrails/readonly-state.ts`, `extensions/guardrails/index.ts` (CLI flag + command), hook integration | ✅ Clean — fits as a feature in `extensions/guardrails/` or standalone extension |

**Priority assessment:** xz-dev's `ignoredBashArgs` is the single highest-value feature across all forks — it directly addresses the #1 complaint about false-positive path prompts. ~200 lines of well-tested, same-architecture code.

---

### Cluster C: "Remember My Allow Decisions"

**The pain point:** I approved something. Now I'm in another session (or another project) and I have to approve it again. Grant persistence is limited to in-memory session scope.

**Forks that hit this:**

| Fork | Feature | Persistence scope |
|------|---------|-------------------|
| AlainS7 | Global path-access grants (hardcoded) | global (~/.pi/agent/extensions/) |
| yonilerner | Configurable `alwaysScope` | local | global (config option) |
| ryanh-ai | Permission gate project/global grants | local | global (keybindings) |
| nikitakot | allow-session in policy confirmation | memory (same session) |

**Semantic convergence:** The core pattern is identical across all four — "I allowed X, save that decision to a scope broader than this session." The differences are:
- **Which feature** (path-access vs permission-gate vs policies)
- **Which scope** (memory → local → global)
- **How it's triggered** (config option vs keybinding vs implicit)

yonilerner's approach (configurable `alwaysScope` option) is the cleanest design of the four — it makes the scope a declarative choice rather than hardcoding or adding UI. But ryanh-ai's approach (keybinding per-save) gives finer control.

**Architecture surface area:**

| What | Files touched | Our fit |
|------|---------------|---------|
| Grant scope type expansion | `extensions/path-access/grants.ts` (add `"global"` to `GrantScope`), `extensions/permission-gate/grants.ts` (same) | ✅ Trivial |
| Config `alwaysScope` option | `src/shared/config/types.ts` (+field), `src/shared/config/defaults.ts` (+default) | ✅ Trivial |
| Scope resolution in prompt result handler | `extensions/path-access/index.ts` (use `alwaysScope` instead of hardcoded `"local"`), `extensions/permission-gate/index.ts` (same) | ✅ Low effort |
| Settings UI | `extensions/guardrails/commands/settings/index.ts` (+dropdown) | ✅ Standard pattern |

**⚠️ Watch item:** The three features (path-access, permission-gate, policies) each have their own grant persistence system, but the concept is the same. If we implement this cluster, consider a **unified grant persistence layer** that all three can call — rather than duplicating the `"global"` scope logic across three files.

---

### Cluster D: "Security Boundaries Have Gaps"

**The pain point:** The existing protections have known escape vectors — symlinks, extensions directory writes, git hook bypasses. Users (or hostile models) can work around guardrails.

**Forks that hit this:**

| Fork | Feature | What it prevents |
|------|---------|-------------------|
| AlainS7 | Symlink-aware boundary checks | Symlink escape (symlink in allowed dir → write outside) |
| AlainS7 | Extension edit gate | Write/edit to `~/.pi/agent/extensions/` without approval |
| phulot | Pre-commit hook bypass detection | `git commit --no-verify`, `HUSKY=0`, etc. |
| prullanferragut | Git push --force regex fix | `git push --force` not caught when flag is at end |

**Semantic convergence:** These are all "hardening" features — addressing specific attack vectors rather than general UX improvements. They don't share a common mechanism but they share a goal: **close specific bypass paths.**

**Architecture surface area:**

| What | Files touched | Our fit |
|------|---------------|---------|
| Symlink-aware boundary | `src/core/paths/path.ts` (+`resolvePathWithSymlinks`, `isWithinBoundaryResolved`), `src/core/paths/access.ts` (+async `checkPathAccess`) | ✅ Clean but has latency implications (async I/O on hot path) |
| Extension edit gate | `extensions/guardrails/hooks/` (new `extension-edit-gate.ts`) or new extension | ✅ Clean — fits policy-based architecture or dedicated rule |
| Pre-commit bypass detection | `extensions/permission-gate/rules.ts` (+`checkPreCommitBypass` function) | ✅ Clean — pure function integration into existing rule pipeline |
| Git push --force regex | `extensions/guardrails/commands/settings/examples.ts` (one-line change) | ✅ Already approved in audit-findings-2.md |

**Priority:** These are valuable but lower urgency than Clusters A/B/C. The extension edit gate is the most impactful (prevents model self-modification). The symlink-aware check is the most technically involved.

---

### Cluster E: "Tell Me What Happened" (Visibility & Feedback)

**The pain point:** When guardrails blocks something, the user and model don't get enough context about why. The UI is opaque, the deny message is generic, and there's no audit trail visible in the prompt.

**Forks / proposals that hit this:**

| Source | Feature | What it exposes |
|--------|---------|-----------------|
| harrisony | `action:prompted` event (already in our repo) | Fired before prompt — extension consumers can observe |
| comment design proposal | `comment` field on patterns | Static guidance text shown above prompt |
| **You (this discussion)** | Trigger indicator at prompt top | Dynamic: shows which rule/pattern actually matched |
| Existing code | `safety.metadata.description` | Already tracked in metadata but not visually prominent |

**Semantic convergence:** Three different approaches to the same problem — the user and model need to know *what* triggered. The approaches differ in audience (user vs model) and mechanism (static text vs dynamic data vs event bus), but the underlying need is the same.

**Your feedback is the strongest design here** because:
1. It uses data already in the system (no new config fields needed)
2. It serves the human user directly (auditability at a glance)
3. It's purely a UI rendering change — zero schema impact, zero migration

**Architecture surface area:**

Already detailed in Section 0 above. Summary:
- `extensions/permission-gate/prompt.ts` — surface `safety.metadata` in header
- `extensions/path-access/prompt.ts` — surface match reason in header
- `extensions/guardrails/index.ts` — surface `PolicyMeta` in block notification

---

## 2. Cross-Cluster Patterns & Deeper Convergence

Beyond the five clusters, there are **higher-order patterns** that span multiple clusters:

### 2.1 The Shared UI Component Problem

nikitakot's `createConfirmationUI` is the only fork that extracted a **shared TUI component** usable by both permission-gate and policies. Currently in our architecture:
- `extensions/permission-gate/prompt.ts` — own prompt component
- `extensions/path-access/prompt.ts` — own prompt component  
- `extensions/guardrails/hooks/policies.ts` — no prompt at all (silent block)

If we implement Cluster A (interactive override), Cluster B (prompt timeouts), AND Cluster E (trigger indicator), we'll be modifying the prompt rendering in three places with similar logic three times. **At that point, extracting a shared prompt utility becomes worthwhile.**

Suggested location: `src/shared/tui/prompt.ts` or `src/shared/ui/`

### 2.2 The Unified Persistence Question

Clusters A and C both need grant persistence:
- Cluster A: "allow-session" → memory scope (already exists)
- Cluster C: "allow-project" → local scope, "allow-global" → global scope

Currently:
- `extensions/path-access/grants.ts` — `persistGrant(path, scope)`
- `extensions/permission-gate/grants.ts` — `saveCommandSessionGrant(command)`
- Policies — no grant persistence at all

If we add project/global scopes to all three, we'll have three implementations of the same config-scope-write pattern. Consider extracting a shared `configSaver` utility that wraps `configLoader.save()`.

### 2.3 The Metadata Pipeline

Most features across all clusters depend on the same pipeline:
```
Config → compile → match → metadata → prompt → result → persistence
```

The features all hook into **different stages** of this pipeline:

| Feature | Stage | Hooks into |
|---------|-------|------------|
| `ignoredBashArgs` (xz-dev) | Pre-compile: arg extraction | `extractBashPathCandidates` |
| `requireConfirmation` (qw457812) | Post-match: before block | Policy hook loop |
| `strictAllowSession` (nikitakot) | Post-match: session grant save | `saveCommandSessionGrant` |
| Prompt timeouts (JJGO) | Prompt: rendering + interaction | Prompt component |
| Read-only bash detection (AlainS7) | Post-match: bypass | Policy hook |
| Trigger indicator (you) | Prompt: rendering | Prompt component |
| Global grants (AlainS7/yonilerner) | Post-result: persistence | Grant handler |

This means the features are **mostly orthogonal** — they touch different parts of the pipeline and don't conflict. The main risk is cumulative complexity if too many features all modify the same prompt component.

---

## 3. Implementation Ordering Recommendation (If We Were to Implement)

Based on convergence strength, effort, and user impact:

| Priority | Feature | Cluster | Effort | Why this rank |
|----------|---------|---------|--------|---------------|
| 1 | **Trigger indicator** (your feedback) | E | Low (~30 lines) | Zero new fields, pure rendering, high auditability value |
| 2 | **`ignoredBashArgs`** (xz-dev) | B | Low (~150 lines) | Highest user-impact feature in all forks. Same architecture, 1:1 file mapping. |
| 3 | **`askConfirmation`** on policies (nikitakot/qw457812) | A | Medium (~200 lines) | Addresses a top complaint (silent blocks). Has good prior art. |
| 4 | **Prompt timeouts** (JJGO) | B | Medium (~300 lines) | Good UX improvement, shared with #3 (prompt infrastructure) |
| 5 | **Extension edit gate** (AlainS7) | D | Medium (~250 lines) | Security hardening — prevents model self-modification |
| 6 | **Global grant scope** (yonilerner) | C | Low (~50 lines) | Small add-on once we have the persistence infrastructure from #3 |
| 7 | **Strict allow-session** (nikitakot) | C | Low (~10 lines) | Tiny, prevents substring-matching accidents |
| 8 | **Path filter** on policies (nikitakot) | A | Medium (~120 lines) | Powerful but more config-complex |
| 9 | **Read-only mode** (nikitakot) | B | Medium (~200 lines) | Opinionated feature, lower priority than ad-hoc noise reduction |
| 10 | **Symlink-aware boundaries** (AlainS7) | D | Medium (~100 lines) | Important but async I/O in hot path needs careful design |
| 11 | **Pre-commit bypass** (phulot) | D | Low (~60 lines) | Narrow use case, but pairs well with #3 |
| 12 | **Read-only bash detection** (AlainS7) | B | Medium (~180 lines) | Overlaps somewhat with `ignoredBashArgs` approach |
| 13 | **Persistent permission gate grants** (ryanh-ai) | C | Medium (~150 lines) | Use-case dependent — less critical than path-access persistence |

**Total cumulative effort** for all 13: ~1,800 lines across ~25 files.

---

## 4. Architecture Tensions to Watch

### 4.1 Prompt Component Bloat

The `permission-gate/prompt.ts` component currently renders a single TUI view. If we add:
- Trigger indicator (Cluster E)
- Prompt timeouts with live countdown (Cluster B)
- `comment` display (if we go back to that)
- Sub-prompt for grant scope selection (ryanh-ai)

The component becomes unwieldy. At some point, a **builder pattern** or **composition approach** would be warranted (e.g., `buildPromptSections() → Section[]` that are rendered in order).

### 4.2 The Config Types Creep

Every feature that adds config fields touches `src/shared/config/types.ts`. Current count of features that would add fields:
- `comment?` — 2 fields
- `IgnoredBashArgRule` — 1 interface
- `askConfirmation` / `requireConfirmation` — 1 field
- `prompts.timeoutSeconds` — 1 field
- `alwaysScope` — 1 field
- `strictAllowSession` — 1 field
- `pathFilter` — 1 field
- `readOnlyMode` — 1 interface

Without discipline, `types.ts` becomes a kitchen sink. Consider **feature-scoped config interfaces** (e.g., `PermissionGateAdvancedConfig`) rather than adding every new field to the top-level config shape.

### 4.3 The Hook Integration Pattern

Features in Clusters A, B, and D all need to modify the behavior of policy/permission-gate hooks. Our current hook pattern is:

```ts
export function setupXxxHook(pi: Pi): Teardown {
  // 1. Create rule
  // 2. Register checkAction handler  
  // 3. In handler: checkAction → prompt → result
}
```

If multiple features need to modify step 2 or 3, we'd either need:
- **Config-driven branching** (preferred): The hook reads config to decide behavior (e.g., `if (config.policies.askConfirmation) { ... }`)
- **Plugin hooks** (over-engineered for now): Allow extensions to register pre/post handlers

Config-driven branching is simpler and matches our existing pattern. It does mean `extensions/guardrails/index.ts` and `extensions/permission-gate/index.ts` become the integration points for most features.

### 4.4 Test Harness Compatibility

All features in the audit come with tests. Our `pi-test-harness.ts` and `pi-context.ts` test utilities were designed for exactly this purpose — testing hooks with mock `pi` instances. Any feature we port should include tests that use these utilities, not the old `__tests__/` patterns from the forks.

---

## 5. Summary: What I'd Grill Each Cluster On

If we were to start implementing any of these, here are the questions I'd push on:

### Cluster A (Interactive Override)
- **Should `askConfirmation` be per-rule or a global policy setting?** qw457812 does per-rule; nikitakot does per-rule. Per-rule is more flexible but adds config surface. Ask: when would you want ALL policies to be confirmable vs just some?
- **The `allow-session` grant in policy confirmation adds a path to the rule's `allowedPatterns`** — but that mutates the compiled config at runtime. How does this interact with config reload?
- **If a user confirms a blocked action, and then the same action happens again, should it re-prompt?** qw457812 says yes (no session tracking for blocked-then-allowed). Is that the right behavior?

### Cluster B (Noise Reduction)
- **`ignoredBashArgs` rules vs `isClearlyReadOnlyBashCommand`** — they overlap in purpose (reduce false positives). Do we need both, or does `ignoredBashArgs` cover the read-only case too? (Answer: no — `ignoredBashArgs` filters before boundary check, `isClearlyReadOnlyBashCommand` bypasses policy enforcement. Different stages.)
- **What's the default timeout for prompts?** JJGO uses 300s. Too short? Too long? Should it be configurable at the prompt level or globally?
- **Read-only mode**: Is this a feature we'd actually use, or does it solve a problem we don't have? nikitakot's implementation is clean, but the use case ("I want to browse, not modify") might be addressed by Clusters B's other features.

### Cluster C (Grant Persistence)
- **"global" scope semantics**: In our config architecture, does "global" mean `~/.pi/agent/extensions/guardrails.json` or something else? Need to verify `configLoader.save('global', ...)` behavior.
- **Risk of accidental permanent allow**: ryanh-ai's approach has a sub-prompt (exact vs class) which adds friction — intentional friction. Is that the right UX? Should there be a confirmation step?

### Cluster D (Security Hardening)
- **Symlink-aware boundary**: The async I/O latency concern is real. `resolvePathWithSymlinks` does up to `path.segment` fs calls. For deep paths, this could add milliseconds. Is that acceptable on every path access check?
- **Extension edit gate**: Should this be in `extensions/guardrails/` or a separate extension? If it's in guardrails, it's self-protecting (guardrails protects its own config). If it's separate, it's more modular but can the model remove it?
- **Pre-commit bypass detection**: The `-n` short flag overlap with `git push -n` (dry run) is a known false positive. phulot's code checks `n in shortFlags` which catches dry-run as well. Worth fixing during port.

### Cluster E (Visibility)
- **What should the trigger indicator show?** Just the rule ID? "admin-commands"? Or the full pattern? "npm install"? There's a tradeoff between informativeness and screen clutter.
- **Should the trigger indicator be shown before or after the command preview?** Before (as a header) gives immediate context. After (as a footnote) reads as explanation. Your suggestion puts it at the top, which matches our existing "Dangerous Command Detected" header pattern — just make it more specific.
- **Does this interact with the `comment` feature, or replace it?** Based on your feedback, this *replaces* the need for a static `comment` field. The trigger IS the comment, dynamically.

---

## Appendix: Quick-Reference Map

```
Feature                        │ Pipeline Stage        │ Config? │ Files Touched
───────────────────────────────┼───────────────────────┼─────────┼─────────────────────
Trigger indicator (your idea)  │ prompt rendering      │ No      │ prompt.ts (x3)
ignoredBashArgs (xz-dev)       │ arg extraction        │ Yes     │ types, bash-paths, targets
askConfirmation (qw/nikitakot) │ post-match, pre-block │ Yes     │ types, rules, index
Prompt timeouts (JJGO)         │ prompt interaction    │ Yes     │ prompt.ts (x2), types
Extension edit gate (AlainS7)  │ pre-tool execution    │ No      │ new file in hooks/
Global grants (yonilerner)     │ post-result persist   │ Yes     │ grants.ts, types, index
Strict allowSession (nikitakot)│ session grant save    │ Yes     │ types, matching, index
Path filter (nikitakot)        │ policy rule matching  │ Yes     │ types, matching, path.ts
Read-only mode (nikitakot)     │ global mode switch    │ Yes     │ new extension file
Symlink boundaries (AlainS7)   │ path checking         │ No      │ path.ts, access.ts
Pre-commit bypass (phulot)     │ dangerous matching    │ No      │ rules.ts (permission-gate)
Read-only bash det. (AlainS7)  │ policy bypass         │ No      │ new util + hook bypass
Persistent perm-gate (ryanh-ai)│ grant persistence     │ Yes     │ grants.ts, index, prompt
```

**Key insight:** 8 of 13 features need config type changes. 4 are pure rendering/logic with zero config impact. The config-heavy features benefit from being implemented as a batch to avoid `types.ts` churn.
