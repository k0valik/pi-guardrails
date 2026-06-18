# Feature Design: `comment` Field (Model Guidance) + Trigger Indicator (UI)

> **Status:** Design proposal - updated 2026-06-18 per discussion
> **Requested by:** k0valik
> **Date:** 2026-06-17 (updated 2026-06-18)

## Problem

When the permission gate or policies block a command, two audiences suffer:

1. **The human user** sees a generic prompt with no indication of *which* guardrail
   rule caught the action. No quick way to assess risk vs false positive.
2. **The model** receives a generic denial message like `"User denied dangerous
   command"` with no guidance on what to do instead.

### Concrete example

User has `npm install` in the permission gate patterns with a custom note:
"> Use `pnpm install` instead"

**Currently (UI):**
1. User sees the prompt: "Dangerous Command Detected - dependency installation"
2. User has to inspect the command to figure out which rule matched
3. User presses `n` (deny)

**Currently (model):**
1. Model receives: `"User denied dangerous command"`
2. Model has no idea why, tries `npm install` again, or tries `npm i` thinking that'll work

**Desired (UI):**
1. User sees at the top of the prompt: ⚠ "Rule: dependency installation | Pattern: npm install"
2. User immediately knows which guardrail caught it and can assess
3. User presses `n` (deny) if appropriate

**Desired (model):**
1. Model receives: `"Blocked by rule: dependency installation. Use pnpm install instead"`
2. Model understands the constraint and switches to `pnpm install`

## Two-Audience Design Summary

| Channel | Shows | Audience | Config change? |
|---------|-------|----------|----------------|
| **UI prompt** | The **trigger** - which rule/pattern matched | Human user | ❌ No - uses existing `safety.metadata` |
| **Deny message** | The **`comment`** - guidance text like "Use pnpm instead" | Model/LLM | ✅ Yes - new optional `comment` field |

These are complementary. The trigger helps the user assess the prompt; the comment
helps the model correct its behavior. The original design proposal mixed both into
the `comment` field - this update separates them.

---

## Existing Mechanisms (What Already Works)

The current codebase already has a `description` field on patterns and rules:

- `DangerousPattern.description` - shown in the prompt (`"This command contains {description}"`)
- `PatternConfig.description` - used by `formatAutoDenyReason()` → `"Command auto-denied: {description}"`
- `PolicyRule.blockMessage` - used as the block reason in policies

But these serve a different purpose:

| Field | Where shown | Where sent on deny |
|-------|-------------|--------------------|
| `description` | In the UI prompt header | **Nowhere** - replaced by `"User denied dangerous command"` |
| `blockMessage` | In the policy hook's notify | Used as-is |
| {{trigger}} (proposed) | **Top** of UI prompt | **Nowhere** - human-only |
| `comment` (proposed) | **Not shown** in UI prompt | Sent to model on deny |

None of the 12 forks analyzed in the audit cover this feature pairing. It's novel.

## Proposed Solution

### 1. New `comment` field on pattern/rule config types

Add an optional `comment` string to all config types that can cause a block:

```ts
// src/shared/config/types.ts

export interface PatternConfig {
  pattern: string;
  description?: string;   // existing: shown in prompt header
  regex?: boolean;
  comment?: string;       // NEW: sent to model on deny, not shown in UI
}  

export interface PolicyRule {
  id: string;
  name?: string;
  description?: string;   // existing
  patterns: PatternConfig[];
  allowedPatterns?: PatternConfig[];
  protection: Protection;
  onlyIfExists?: boolean;
  blockMessage?: string;  // existing: used as block reason
  comment?: string;       // NEW
  enabled?: boolean;
}
```

`DangerousPattern extends PatternConfig`, so it inherits `comment` for free.

### 2. Display the trigger in the TUI prompt (human user)

When the prompt fires, the top line shows what *caught* the action - sourced from
metadata the system **already has** (`safety.metadata.description`,
`safety.metadata.pattern`, `safety.reason`):

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

Implementation in `extensions/permission-gate/prompt.ts`:

- Reorder the prompt component to render `safety.metadata` info at the top,
  before the command preview
- Show rule name (from `metadata.description` or auto-labelled)
  and the specific pattern that matched (`metadata.pattern`)
- No new parameters needed - the metadata is already passed through

For path-access (`extensions/path-access/prompt.ts`):

- Show the command + target path that triggered the boundary check
- Surface the matched `allowedPaths` entry if relevant

For policies (`extensions/guardrails/index.ts`):

- When `checkAction` returns a match, include the `ruleId` + `protection` in
  the block notification so the user sees which policy rule fired

This is **purely a rendering change** - the data already flows through
`safety.metadata`. No new config fields, no migration, no schema update.


### 3. Return comment to the model on deny

**Permission gate** (`extensions/permission-gate/index.ts`):

Replace the hardcoded `"User denied dangerous command"` (line 124) with:

```ts
const denyReason = safety.metadata?.comment
  ? `Blocked by rule: ${safety.metadata.description}. ${safety.metadata.comment}`
  : `User denied dangerous command: ${safety.reason}`;
```

**Auto-deny** (`extensions/permission-gate/rules.ts`):

Update `formatAutoDenyReason` to prefer `comment` over `description`:

```ts
export function formatAutoDenyReason(pattern: PatternConfig): string {
  if (pattern.comment) return pattern.comment;
  const description = pattern.description?.trim();
  return description
    ? `Command auto-denied: ${description}`
    : "Command matched auto-deny pattern and was blocked automatically.";
}
```

**Policies** (`extensions/guardrails/index.ts` + `rules.ts`):

- Add `comment?: string` to `CompiledPolicy` and `PolicyMeta`
- In `createPolicyRules`, thread `policy.comment` into metadata
- In `setupPolicyHook`, use `safety.metadata?.comment` if present to build
  the block reason


### 4. Thread comment through the metadata pipeline

The key is getting the `comment` from config → compiled rule → match metadata →
block reason.

```
PatternConfig.comment
    ↓
compileCommandPatterns → CompiledPattern.source.comment (already on PatternConfig)
    ↓
matchCommandPattern returns PatternConfig (has .comment)
    ↓
createPermissionGateRule returns Rule<PermissionGateMeta>
    ↓
PermissionGateMeta includes .comment
    ↓
On deny: safety.metadata.comment → denyReason
```

Currently `PermissionGateMeta` is:

```ts
type PermissionGateMeta = {
  command: string;
  description: string;
  pattern: string;
};
```

Add: `comment?: string;`

Similarly `PolicyMeta`:

```ts
type PolicyMeta = {
  ruleId: string;
  protection: Protection;
  path: string;
};
```

Add: `comment?: string;`

### 5. Settings UI integration

The pattern editor component (`extensions/guardrails/components/pattern-editor.ts`)
currently has a three-field form: `pattern` + `description` + `regex`. Add an
optional fourth field `comment`:

- Label: "Deny feedback (visible to the model)"
- Description: "What the model sees when this is denied — e.g., 'Use pnpm instead'"
- Optional, defaults to empty

For policy rules, add a `comment` field to the rule editor in
`extensions/guardrails/commands/settings/index.ts`, similar to the existing
`blockMessage` field.

### 6. Built-in defaults

The built-in default patterns in `src/shared/config/defaults.ts` could get
`comment` entries for the most common user friction points:

```ts
{
  pattern: "npm install",
  description: "dependency installation",
  comment: "Use pnpm instead of npm for all package management in this project."
}
```

This would be the `applyBuiltinDefaults` path — users who opt-in get useful
deny feedback out of the box.

## Optional: Interactive "Deny with Reason"

As an enhancement, add a **deny-with-reason** prompt option (e.g., `d` key) that
opens a text input for the user to type a custom reason:

```
y/enter: allow • a: session • d: deny with reason • n/esc: deny
```

When `d` is pressed:
1. The prompt switches to a text input: `"Reason for denial (what should the model do instead?):"`
2. User types a message (e.g., "use pnpm instead")
3. The typed message is returned as the block reason

This would co-exist with the `comment` field: if a comment exists, it's used as
the prompt text for the input, pre-filling it as a suggestion.

```
Reason for denial (what should the model do instead?):
> Use pnpm instead of npm                  ← pre-filled from comment
  [submit]  [cancel]
```

Implementation: requires a mode switch in `createPermissionGateConfirmComponent`
— from confirmation mode to text-input mode. The TUI already supports `Input`
components (see `pattern-editor.ts` usage).

## Call Sites Summary

| File | Change |
|------|--------|
| `src/shared/config/types.ts` | Add `comment?: string` to `PatternConfig`, `PolicyRule` |
| `src/shared/config/defaults.ts` | Add `comment` to built-in patterns where useful |
| `src/shared/matching.ts` | Thread `comment` through pattern compilation (already on `PatternConfig`) |
| `extensions/permission-gate/rules.ts` | Add `comment` to `PermissionGateMeta`; update `formatAutoDenyReason` |
| `extensions/permission-gate/prompt.ts` | **(Trigger indicator)** Reorder to show `safety.metadata` at top - no `comment` rendering here |
| `extensions/permission-gate/index.ts` | Use `safety.metadata.comment` in deny reason |
| `extensions/guardrails/rules.ts` | Add `comment` to `CompiledPolicy`, `PolicyMeta`; thread through `createPolicyRules` |
| `extensions/guardrails/index.ts` | Use `safety.metadata.comment` in block reason for policies |
| `extensions/guardrails/components/pattern-editor.ts` | Add `comment` field to pattern editor form |
| `extensions/guardrails/commands/settings/index.ts` | Add `comment` field to policy rule editor |
| `schema.json` | Update JSON schema with `comment` field |


## Prior Art in Forks

None of the 12 forks analyzed cover this feature pairing (trigger + comment).
Closest related work:

- **qw457812** / **nikitakot** - `requireConfirmation`/`askConfirmation` on policies
  adds an interactive confirmation dialog, but the deny message is still generic.
  The trigger indicator would surface in their `createConfirmationUI` naturally.
- **phulot** - auto-deny patterns with descriptive messages (`formatAutoDenyReason`
  already exists). The `comment` field extends this with model-facing feedback.
- **prullanferragut** - example pattern fix. If we add `comment` to built-in
  defaults, the examples should also include comments.


## Relationship to Existing Features

| Existing | Feature | Relationship |
|----------|---------|--------------|
| `description` on patterns | `comment` on patterns | Complementary. `description` labels the risk in the prompt header; `comment` sends model-facing guidance on deny. |
| `description` on patterns | Trigger indicator | Same source field - the `description` / `pattern` from `safety.metadata` is surfaced at the top of the UI prompt. |
| `blockMessage` on PolicyRule | `comment` on PolicyRule | `blockMessage` is the default block reason; `comment` supplements it with model-facing guidance. |
| `formatAutoDenyReason(description)` | `formatAutoDenyReason(comment?)` | `comment` takes priority over `description` for auto-deny. |
| `requireConfirmation` (qw457812) | Trigger indicator + comment | The confirmation dialog's subtitle area shows the trigger; the deny message includes the comment. |
| Built-in defaults | comment in defaults | Adding comments to default rules makes them more usable out of the box. |


## Effort Estimate

| Component | Effort | Dependencies |
|-----------|--------|-------------|
| Config types (`comment` field) | Low (~5 lines) | None |
| Permission gate metadata + deny reason | Low (~15 lines) | Config types |
| Auto-deny reason | Low (~5 lines) | Config types |
| Trigger indicator (permission gate prompt) | Low (~10 lines) | None - uses existing metadata |
| Trigger indicator (path-access prompt) | Low (~10 lines) | None - uses existing metadata |
| Policy comment integration | Low (~15 lines) | Config types |
| Settings UI pattern editor | Low (~20 lines) | None |
| Settings UI rule editor | Low (~20 lines) | None |
| Built-in defaults | Low (~15 entries) | Config types |
| **Subtotal (core)** | **~100 lines** | |
| Deny-with-reason interactive input | Medium (~60 lines) | Core implementation |
| Schema update | Low (~10 lines) | Config types |
| **Total** | **~170 lines** | |


## Future Considerations

1. **`comment` on `allowedPatterns`** — If a user adds an allowed pattern exception
   with a comment, the model would see the comment if the exception is later
   revoked or if a sibling pattern matches.

2. **`comment` on `autoDenyPatterns`** — Currently `PatternConfig` handles this
   via the shared `description`→`comment` inheritance. No additional work needed.

3. **Multi-lingual comments** — Could support a `comment` map per locale, but
   over-engineering for now.

4. **Comment variables** — Like `blockMessage` supports `{file}` placeholder,
   comments could support `{command}` or `{pattern}` to make dynamic messages:
   `"Use pnpm add {package} instead of npm install"`.
