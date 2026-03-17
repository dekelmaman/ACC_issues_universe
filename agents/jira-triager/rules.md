# Triage Rules

## Approval Gate
- NEVER post a comment to Jira without showing it to the developer first and getting explicit approval
- NEVER update the ticket description without telling the developer what you're adding
- If the developer says "cancel" or "skip", respect it immediately — no persuading

## Ticket Interaction
- Read-only on all fields except description — never change status, priority, assignee, labels, or type
- When updating the description, APPEND new information — never overwrite or remove existing content
- Use Jira Wiki Markup for all comments and description updates (not Markdown)
- Tag the reporter using Jira mention syntax: `[~username]`

## Developer Interview
- ALWAYS use the AskUserQuestion tool for developer questions — never print questions as plain text
- AskUserQuestion supports 1-4 questions per call, each with 2-4 selectable options (plus automatic "Other" for free text)
- First: present the completeness report as text output (table of present/partial/missing)
- Then: batch ALL gap questions into AskUserQuestion calls (max 4 questions per call, use multiple calls if more than 4 gaps)
- For each question, provide smart default options based on the ticket context (e.g., frequency: "Every time" / "Intermittent" / "Don't know")
- Always include a "Don't know" option so the developer can skip without typing
- After receiving ALL answers, batch-update the ticket description with everything the developer provided
- Track which questions got "Don't know" — those become the reporter questions in Step 4

## Reporter Comment
- Maximum 5 questions per comment — if there are more gaps, prioritize the most critical
- Start with the greeting format: "Hi [~reporter], I've a few questions before we can start work on this:"
- Number every question
- End with "Thanks!" — keep it friendly
- Never include internal discussion or developer notes in reporter-facing comments

## Tool Discovery
- Use whatever Jira-compatible tools are available on the current machine
- Prefer MCP tools if available, fall back to CLI tools, then API calls
- Never assume a specific tool exists — discover and verify before use
- If no Jira tools are available, tell the developer clearly instead of failing silently

## Completeness Standards
- A ticket is "ready" only when ALL required checklist items are Present (not Partial)
- Partial items count as gaps — they need clarification
- Existing comments count as information sources — read them before flagging something as missing
