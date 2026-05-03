- Never modify code — audit only, propose fixes
- Always include full file paths with `GGIssueDomain/...` namespacing (never just "MainView")
- Include line numbers for every finding
- Present findings in tables with: File | Line | Severity | Issue | Fix
- Flag AlloyUI gaps separately (native is acceptable when no AlloyUI equivalent exists)
- Check both `import AlloyUI` presence AND actual component usage
- Detect UIColor leaks in domain/model layers
- When multiple features share similar view names, always use the full path to disambiguate

## AlloyUI Gaps (native acceptable)
- DatePicker — no AlloyUI equivalent
- VStack / HStack / ZStack — layout scaffolding
- .clipShape() — no AlloyUI modifier
- Alert / Sheet — no AlloyUI equivalent
- NavigationLink — no AlloyUI equivalent

## Missing-Token Reporting (MANDATORY)

When a finding's recommended fix would require a token that does NOT exist in AlloyUI:

- DO NOT propose an arbitrary substitute hex / asset / font fallback.
- DO mark the finding with severity `🟠 NEEDS_NEW_TOKEN` (distinct from 🔴 violation).
- DO state plainly: "Fix requires `<token-name>` which is not in AlloyUI. Adding it
  needs user approval — see Trinity's design-system-asset-protocol."
- DO show the closest existing alternative + which file/line it lives in, so the user
  can pick "add new" vs "use alternative" later.

Severity ladder:
- 🔴 VIOLATION — AlloyUI equivalent exists, code uses native primitive
- 🟠 NEEDS_NEW_TOKEN — fix needs a token not yet in the design system
- 🟡 WARNING — unclear / ambiguous mapping
- ⚪ CONSISTENCY GAP — sibling view uses different token for the same KIND of element
- ✅ PASS

## Cross-Feature Consistency (MANDATORY)

In addition to the file-by-file compliance pass, do a cross-feature consistency check:

1. **Scope** — the feature folder containing the primary file under audit, plus 1-2
   adjacent feature folders the user lists. Do NOT scan the whole repo.
2. **Pattern matching** — for each KIND of element in the audited file (CTA buttons,
   AI surfaces, primary actions, microphone icons, sparkle indicators, etc.), search
   the scope for OTHER instances and compare their tokens.
3. **Report** — list every divergence as a ⚪ CONSISTENCY GAP finding:
   ```
   File | Line | Element kind | Audited file uses | Sibling uses | Suggested action
   ```
4. The fix may be "update sibling to match" or "no change — sibling has different
   purpose". You PROPOSE; the human decides.

The point is: visual consistency in a feature suite (e.g. "all AI surfaces use
`purple500`") is just as important as native-vs-Alloy compliance, but easier to miss
because every individual file may pass on its own.
