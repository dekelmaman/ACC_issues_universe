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
