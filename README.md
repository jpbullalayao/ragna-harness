# ragna-ai

Personal agent skills for my own workflows. Each skill is individually installable via the [skills.sh](https://skills.sh) CLI.

## Skills

### `/self-code-review`

Reviews the current branch's diff against `origin/main`. Flags functionality regressions, unnecessary React hooks, duplicated helpers/components, and stale leftover code. Use before opening a PR or when asked to "review my changes".

```bash
npx skills add jpbullalayao/ragna-ai --skill self-code-review
```

### `/submit-code-review`

Posts the findings from `/self-code-review` as GitHub PR comments via the `gh` CLI. Inline comments are preferred (attached to the specific file and line); falls back to regular PR conversation comments. Run `/self-code-review` first, then `/submit-code-review`.

```bash
npx skills add jpbullalayao/ragna-ai --skill submit-code-review
```

### `/submit-pull-request`

Creates a GitHub PR from the current branch using a fixed template (Ticket, Problem, Solution, Before, After, Test plan). Use when you want to "open a PR", "create a pull request", or "push this up as a PR".

```bash
npx skills add jpbullalayao/ragna-ai --skill submit-pull-request
```

### `/code-cleanup [<branch>]`

Analyzes the current branch's diff against `origin/main` (or a specified branch) and auto-applies cleanup fixes across three areas: code brevity & quality, regression risks, and CI/build health. For large diffs, invokes `/simplify` first; for React files, invokes `/react-doctor` before applying fixes. Each finding is classified as `[AUTO]` (applied immediately), `[ARCH]` (architectural improvement — shown as before/after, then applied), or `[MANUAL]` (surfaced for human review, not touched). Runs type checks and build verification after each pass.

```bash
npx skills add jpbullalayao/ragna-ai --skill code-cleanup
```

### `/create-ticket`

Creates a ticket in your preferred issue tracker (defaults to Linear) with a consistent two-section format designed to be clear to both humans and AI agents. The **Context** section explains how the current system works; the **Requirements** section lists actionable acceptance criteria. Explores the codebase to ground the Context in real implementation details. Use when you want to "create a ticket", "file an issue", "write up a ticket", or "open a Linear issue".

```bash
npx skills add jpbullalayao/ragna-ai --skill create-ticket
```

## Install all at once

```bash
npx skills add jpbullalayao/ragna-ai
```

## Requirements

- `gh` CLI authenticated to GitHub (`gh auth login`)
- Claude Code with MCP Linear server connected:
  - Required for `/create-ticket` (used to create issues)
  - Optional for `/self-code-review` (fetches ticket context — degrades gracefully if unavailable)
