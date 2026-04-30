Run the Rick workflow `bug-fix` from universe `ACC_issues_universe`.

Interpret any user-supplied arguments as workflow input.
- If the input looks like a Jira ticket key such as `SCCOM-12345`, use it as `ticket_key`.
- Otherwise treat the input as the bug `description`.
- Unless the user explicitly says otherwise, use `platform=android`.

If the needed workflow input is missing, ask for it before proceeding.
Do not switch to a different universe or workflow.
