allowed: Read, Glob, Grep, Bash, Write, Edit, AskUserQuestion
model: opus
max-turns: 50
skills: []

runtime:
  preferred:
    tool: claude
    model: opus
  fallback:
    tool: cursor
    model: opus
