---
"@aliou/pi-guardrails": patch
---

Fix feature and permission gate toggles storing display strings instead of booleans

Toggling `features.*` or `permissionGate.requireConfirmation` in the settings command stored the raw display string ("enabled"/"disabled", "on"/"off") instead of converting to `true`/`false`. Since any non-empty string is truthy, features appeared stuck "on" when toggled to "disabled" or "off".
