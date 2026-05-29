---
name: create-ticket
description: Create a well-structured ticket in the user's preferred issue tracker (defaults to Linear). Tickets include a Context section (how the current system works) and a Requirements section (what needs to be done), written for both humans and agents. Use when the user types /create-ticket or asks to "create a ticket", "file an issue", "write up a ticket", "open a Linear issue", or "log this as a ticket".
allowed-tools:
  - "Bash(git log *)"
  - "Bash(git branch *)"
  - "Bash(git status *)"
  - "Bash(git rev-parse *)"
  - "Bash(grep *)"
  - "Bash(find *)"
  - "Bash(ls *)"
  - "Read"
  - "AskUserQuestion"
---

# Create Ticket

Creates a ticket in the user's preferred issue tracker with a consistent two-section format readable by both humans and AI agents. **Defaults to Linear** via the `linear` skill unless the user specifies a different tracker.

## When to invoke

Trigger on any of:
- `/create-ticket`
- "create a ticket" / "file an issue" / "write up a ticket"
- "open a Linear issue" / "open a Jira ticket" / "create a GitHub issue"
- "log this as a ticket" / "track this in Linear"

## Ticket format

Every ticket created by this skill uses this exact description structure, in this order:

```markdown
## Context

<How the current system works today. Relevant files, components, APIs, and data flows. Written so a human or agent reading this cold can understand the existing implementation without guessing.>

## Requirements

<What needs to be done. Acceptance criteria written as clear, actionable items. Specific enough for an agent to execute without ambiguity.>
```

If a section cannot be filled from available context, leave a `<!-- fill in -->` placeholder — never invent content.

## Workflow

### Step 1: Parse intent and choose issue tracker

Extract:
- **Topic/ask** — what the ticket is about. If nothing was passed, ask: "What should this ticket be about?"
- **Issue tracker** — which system to use:
  - If the user explicitly names one (e.g., "Jira", "GitHub Issues", "Shortcut"), use the skill or CLI they specify.
  - If no tracker is named, **default to the `linear` skill**.

### Step 2: Explore the codebase for Context

Use the codebase to ground the Context section in reality. Run in parallel:

```bash
git status
git branch --show-current
git log --oneline -10
```

Then search for files relevant to the ticket topic:

```bash
grep -r "<topic keywords>" --include="*.ts" --include="*.tsx" --include="*.js" -l .
find . -name "<relevant filename pattern>" -not -path "*/node_modules/*"
```

Read key files (entry points, relevant components, API routes) to understand current behavior. Summarize in plain language: how does the existing system work today? Cap reads to 3–5 files to stay efficient.

If the codebase has no clear relevance to the topic (e.g., the ticket is about a process or policy, not code), skip exploration and leave the Context section with a `<!-- fill in -->` placeholder.

### Step 3: Draft the ticket

Produce:

**Title** — concise, verb-first (e.g., "Add CSV export to the reporting dashboard"). Under 70 characters.

**Description** — using the exact two-section template above:
- **Context**: synthesize what you found in Step 2 plus the user's framing. Cover the current behavior, relevant files/components, and any known constraints. Keep it factual — do not speculate.
- **Requirements**: list actionable items derived from the user's ask. Each item must be specific enough for an agent to act on without further clarification. If requirements are vague or absent, leave the section with a `<!-- fill in -->` placeholder.

**Do not pause for review.** Proceed directly to Step 4 after drafting. Missing content gets a placeholder, not a question.

### Step 4: Resolve destination

- **Linear (default):** Invoke the `linear` skill to list available teams. Auto-select if only one team exists; ask the user which team if multiple exist and the correct one isn't obvious from context. Optionally associate with a project if the user mentions one.
- **Other tracker:** Use the skill or CLI the user specified. Follow its conventions for workspace/project/board selection.

### Step 5: Create and return

- **Linear (default):** Invoke the `linear` skill to create the issue using the title, team, and description from Steps 3–4.
- **Other tracker:** Use the corresponding skill or CLI with the same title and description content.

Return the ticket URL and identifier (e.g., `KOR-456`) so the user can open it and complete any placeholder sections.

## Constraints

- **Create immediately.** Never block on user approval — placeholders handle missing content.
- **Ground Context in fact.** Only write what can be verified from the codebase or the user's message. Never invent.
- **Requirements must be actionable.** No vague entries like "improve X". Each item must be executable by an agent or human without further clarification.
- **Placeholders over invention.** When a section can't be filled, use `<!-- fill in -->` rather than fabricating content.
- **One ticket per invocation.** Never create duplicates.
- **Respect the tracker.** If the user specified a non-Linear tracker, use that — do not fall back to Linear.
- **Surface errors clearly.** If the `linear` skill or another tool fails, report the error explicitly; do not silently succeed with partial output.
