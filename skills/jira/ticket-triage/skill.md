# Jira Ticket Triage

Analyze a Jira ticket to determine if it has enough detail to begin work. If not, gather missing information from the developer and the reporter.

## Required Capabilities

- **Read ticket** — Fetch a Jira issue by key (summary, description, type, reporter, comments, priority, labels)
- **Read ticket comments** — Fetch existing comments to avoid asking already-answered questions
- **Update ticket description** — Modify the issue description with new information
- **Post ticket comment** — Add a comment to the issue
- **Ask developer** — Interactive Q&A with the current developer (the human in the conversation)

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Ticket key | Yes | e.g., `SCCOM-12345` |

## Triage Flow

### Step 1: Read & Classify

1. Fetch the ticket (summary, description, type, reporter, priority, labels)
2. Fetch existing comments — previous discussion may already contain answers
3. Classify the ticket type: Bug, Task, Story, Sub-task, etc.
4. Select the appropriate completeness checklist based on type

### Step 2: Evaluate Completeness

Run the ticket against the relevant checklist. For each item, mark it as:
- **Present** — clearly described in the ticket or comments
- **Partial** — mentioned but vague or incomplete
- **Missing** — not mentioned at all

#### Bug Ticket Checklist
- [ ] **Summary** — Clear, specific one-line description of the bug
- [ ] **Steps to Reproduce** — Numbered steps someone else can follow
- [ ] **Expected Behavior** — What should happen
- [ ] **Actual Behavior** — What actually happens (including error messages, screenshots)
- [ ] **Environment** — OS version, app version, device model, build type (debug/release)
- [ ] **Frequency** — Always, intermittent, one-time? Regression or new?
- [ ] **Impact/Severity** — How many users affected? Is there a workaround?
- [ ] **Visual Evidence** — Screenshots, screen recordings, crash logs, stack traces

#### Task/Story Checklist
- [ ] **Goal** — What is being accomplished and why
- [ ] **Acceptance Criteria** — Clear definition of done
- [ ] **Scope** — What's in, what's explicitly out
- [ ] **Dependencies** — Blocked by or blocking other tickets
- [ ] **Design/Specs** — Links to designs, specs, or documentation

### Step 3: Ask the Developer (Interactive Batch)

Before reaching out to the reporter, the developer working the ticket may already know some answers. Use the interactive AskUserQuestion tool — never plain text questions.

1. Present the completeness report as text: what's present, partial, and missing
2. Use AskUserQuestion tool to ask about ALL gaps:
   - Max 4 questions per AskUserQuestion call (tool limit). If more than 4 gaps, use multiple calls.
   - Each question gets 2-4 selectable options tailored to the gap type, plus "Other" is automatic for free text
   - ALWAYS include a "Don't know" option so the developer can skip without typing
   - Example questions with options:

   ```
   Question: "How often does this bug reproduce?"
   Header: "Frequency"
   Options:
     - "Every time" (description: "100% reproducible on every attempt")
     - "Intermittent" (description: "Sometimes happens, not always")
     - "Don't know" (description: "Haven't tested, need reporter input")

   Question: "Do you have visual evidence (screenshots/recordings)?"
   Header: "Evidence"
   Options:
     - "Yes, I can capture it" (description: "I'll reproduce and attach evidence")
     - "No, need from reporter" (description: "Can't reproduce locally, ask reporter")
     - "Don't know" (description: "Haven't tried reproducing yet")
   ```

3. Developer selects options or types free text via "Other" — all answers come back at once
4. Parse responses — separate answered items from "Don't know" items
5. Batch-update the ticket description with ALL answered items at once — append a section:

```
----
h3. Additional Details (added during triage)
* *Steps to Reproduce:* [developer's answer]
* *Environment:* [developer's answer]
```

6. The remaining "Don't know" items become the reporter questions in Step 4

### Step 4: Draft Reporter Comment

If there are still unanswered questions after the developer's input:

1. Draft a comment in this format:

```
Hi [~reporter_username],

I've a few questions before we can start work on this:

1. [First question — specific and actionable]
2. [Second question — specific and actionable]
3. [Third question — specific and actionable]

Thanks!
```

2. **Show the draft to the developer for approval** — never post without explicit confirmation
3. The developer may:
   - **Approve** → post the comment
   - **Edit** → modify questions, then post
   - **Cancel** → skip posting (maybe they'll investigate themselves)

### Step 5: Post & Report

1. If approved, post the comment to the ticket
2. Summarize what was done:
   - What information was added to the description
   - What questions were posted to the reporter
   - What's still needed before work can begin

## Output

A triage summary containing:
- Ticket classification and completeness score
- Items filled in by the developer (with description updated)
- Questions posted to the reporter (if any)
- Readiness verdict: **Ready to work** / **Waiting on reporter** / **Needs investigation**

## Question Writing Guidelines

- Be specific — "What iOS version?" not "What environment?"
- Be actionable — ask for something the reporter can provide
- Reference what you do know — "You mentioned it crashes on the list screen — does it happen on first load or after scrolling?"
- One question per numbered item — don't bundle multiple asks
- Keep the tone professional and friendly — we're asking for help, not interrogating
