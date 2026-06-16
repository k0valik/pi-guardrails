# Feature Design: `comment` Field for Deny Feedback

> **Status:** Design proposal — not yet implemented
> **Requested by:** k0valik
> **Date:** 2026-06-17

## Problem

When the permission gate or policies block a command, the model receives a
generic denial message:

- `"User denied dangerous command"` — when the user presses deny
- `"Command auto-denied: {description}"` — for auto-deny patterns

These messages don't tell the model **why** the command was denied or **what
to do instead**. The model has no guidance on how to correct its behavior.

### Concrete example

User has `npm install` in the permission gate patterns with a custom note:
> "Use `pnpm install` instead"

**Currently:**
1. User sees the prompt: "Dangerous Command Detected — dependency installation"
2. User presses `n` (deny)
3. Model receives: `"User denied dangerous command"`
4. Model has no idea why, tries `npm install` again, or tries `npm i` thinking that'll work

**Desired:**
1. User sees above the prompt: ⚠ "Use `pnpm install` instead"
2. User presses `n` (deny)
3. Model receives: `"Blocked by rule: dependency installation. Use pnpm install instead"`
4. Model understands the constraint and switches to `pnpm install`

## Existing Mechanisms (What Already Works)

The current codebase already has a `description` field on patterns and rules:

- `DangerousPattern.description` — shown in the prompt (`"This command contains {description}"`)
- `PatternConfig.description` — used by `formatAutoDenyReason()` → `"Command auto-denied: {description}"`
- `PolicyRule.blockMessage` — used as the block reason in policies

But these serve a different purpose:

| Field | Where shown | Where sent on deny |
|-------|-------------|--------------------|
| `description` | In the UI prompt header | **Nowhere** — replaced by `"User denied dangerous command"` |
| `blockMessage` | In the policy hook's notify | Used as-is |
| `comment` (proposed) | **Above** the UI prompt | Sent to model on deny |

None of the 12 forks analyzed in the audit cover this feature. It's novel.

## Proposed Solution

### 1. New `comment` field on pattern/rule config types

Add an optional `comment` string to all config types that can cause a block:

```ts
// src/shared/config/types.ts

export interface PatternConfig {
  pattern: string;
  description?: string;   // existing: shown in prompt header
  regex?: boolean;
  comment?: string;       // NEW: shown above prompt, sent on deny
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

### 2. Display comment in the TUI prompt

When a `comment` is present, show it **above** the command/description section
as a styled note box, before the action options:

```
╔══════════════════════════════════════╗
║  ⚠ Use pnpm install instead         ║  ← comment rendered as note
║  ─────────────────────────────       ║
║  Dangerous Command Detected          ║
║                                      ║
║  This command contains dependency    ║
║  installation:                       ║
║                                      ║
║   1│ npm install lodash              ║
║                                      ║
║  Allow execution?                    ║
║  y/enter: allow • a: session •       ║
║  n/esc: deny                         ║
╚══════════════════════════════════════╝
```

Implementation in `extensions/permission-gate/prompt.ts`:

- Add optional `comment?: string` parameter to `createPermissionGateConfirmComponent`
- If comment is present, add a `DynamicBorder(warningColour)` + `Text(comment)` block
  before the command section
- The comment should use `ctx.ui.theme.fg("warning", comment)` or similar distinct
  styling so it visually separates from the description

For policies, the `createConfirmationUI` from nikitakot's fork (see audit
findings, feature 2) already supports a `subtitle` field — `comment` could be
an additional `note` parameter or be rendered into the subtitle area.

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
| `extensions/permission-gate/rules.ts` | Add `comment` to `PermissionGateMeta`; update `formatAutoDenyReason` |
| `extensions/permission-gate/prompt.ts` | Add `comment` param to `createPermissionGateConfirmComponent`; render comment above command section |
| `extensions/permission-gate/index.ts` | Use `safety.metadata.comment` in deny reason; pass comment to prompt |
| `extensions/guardrails/rules.ts` | Add `comment` to `CompiledPolicy`, `PolicyMeta`; thread through `createPolicyRules` |
| `extensions/guardrails/index.ts` | Use `safety.metadata.comment` in block reason for policies |
| `extensions/guardrails/components/pattern-editor.ts` | Add `comment` field to pattern editor form |
| `extensions/guardrails/commands/settings/index.ts` | Add `comment` field to policy rule editor |
| `schema.json` | Update JSON schema with `comment` field |

## Prior Art in Forks

None of the 12 forks analyzed cover this feature. Closest related work:

- **qw457812** / **nikitakot** — `requireConfirmation`/`askConfirmation` on policies
  adds an interactive confirmation dialog, but the deny message is still generic.
  The `createConfirmationUI` from nikitakot's fork provides the right component
  architecture to extend with a comment display.
- **phulot** — auto-deny patterns with descriptive messages (`formatAutoDenyReason`
  already exists). The `comment` field extends this with model-facing feedback.
- **prullanferragut** — example pattern fix. If we add `comment` to built-in
  defaults, the examples should also include comments.

## Relationship to Existing Features

| Existing | This feature | Relationship |
|----------|--------------|--------------|
| `description` on patterns | `comment` on patterns | Complementary. `description` labels the risk for the prompt header; `comment` guides both user and model on alternatives. |
| `blockMessage` on PolicyRule | `comment` on PolicyRule | `blockMessage` is the default block reason; `comment` overrides/supplements it with model-facing guidance. |
| `formatAutoDenyReason(description)` | `formatAutoDenyReason(comment?)` | `comment` takes priority over `description` for auto-deny. |
| `requireConfirmation` (qw457812) | comment in confirmation dialog | The dialog shows the comment above the prompt, and uses it on deny. |
| Built-in defaults | comment in defaults | Adding comments to default rules makes them more usable out of the box. |

## Effort Estimate

| Component | Effort | Dependencies |
|-----------|--------|-------------|
| Config types | Low (~5 lines) | None |
| Permission gate metadata + deny reason | Low (~15 lines) | Config types |
| Auto-deny reason | Low (~5 lines) | Config types |
| Permission gate prompt comment display | Low (~15 lines) | None |
| Policy comment integration | Low (~15 lines) | Config types |
| Settings UI pattern editor | Low (~20 lines) | None |
| Settings UI rule editor | Low (~20 lines) | None |
| Built-in defaults | Low (~15 entries) | Config types |
| **Subtotal (core)** | **~95 lines** | |
| Deny-with-reason interactive input | Medium (~60 lines) | Core implementation |
| Schema update | Low (~10 lines) | Config types |
| **Total** | **~165 lines** | |

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
