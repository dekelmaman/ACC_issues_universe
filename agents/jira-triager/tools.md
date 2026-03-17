# Triage Tools

## Skills
skills: [jira/ticket-triage]

## Tool Discovery

This agent requires Jira integration but does NOT assume a specific provider. At runtime, discover available tools:

### Required Capabilities → Tool Resolution

| Capability | Look for (in order) |
|-----------|---------------------|
| Read ticket | MCP tools matching `*jira*getIssue*` → `jira-cli issue view` → direct API |
| Read comments | MCP tools matching `*jira*getIssueComments*` → `jira-cli issue comments` → direct API |
| Update description | MCP tools matching `*jira*updateIssue*` → `jira-cli issue edit` → direct API |
| Post comment | MCP tools matching `*jira*postIssueComment*` → `jira-cli issue comment add` → direct API |
| Ask developer | Use the AskUserQuestion tool or equivalent interactive prompt |

### Discovery Protocol

1. Check for MCP-provided Jira tools (search available tools for patterns like `jira`, `mcp-jira`)
2. If no MCP tools, check for CLI: `which jira-cli` or `which jira`
3. If no CLI, check for environment variables that suggest API access: `JIRA_BASE_URL`, `JIRA_TOKEN`
4. If nothing found, report clearly: "No Jira tools available on this machine"

### Jira Markup Reference

When writing comments or updating descriptions, use Jira Wiki Markup (Data Center edition):
- Headers: `h3. Title`
- Bold: `*text*`
- Lists: `* item` (unordered), `# item` (ordered)
- Mentions: `[~username]`
- Horizontal rule: `----`
- Code: `{{inline code}}` or `{code}block{code}`
