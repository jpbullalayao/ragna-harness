# ragna-harness

Personal agent skills for my own workflows. Each skill is individually installable via the [skills.sh](https://skills.sh) CLI.

## Skills

### `/self-code-review`

Reviews the current branch's diff against `origin/main`. Flags functionality regressions, unnecessary React hooks, duplicated helpers/components, and stale leftover code. Use before opening a PR or when asked to "review my changes".

```bash
npx skills add jpbullalayao/ragna-harness --skill self-code-review
```

### `/submit-code-review`

Posts the findings from `/self-code-review` as GitHub PR comments via the `gh` CLI. Inline comments are preferred (attached to the specific file and line); falls back to regular PR conversation comments. Run `/self-code-review` first, then `/submit-code-review`.

```bash
npx skills add jpbullalayao/ragna-harness --skill submit-code-review
```

### `/submit-pull-request`

Creates a GitHub PR from the current branch using a fixed template (Ticket, Problem, Solution, Before, After, Test plan). Use when you want to "open a PR", "create a pull request", or "push this up as a PR".

```bash
npx skills add jpbullalayao/ragna-harness --skill submit-pull-request
```

## Install all at once

```bash
npx skills add jpbullalayao/ragna-harness
```

## Requirements

- `gh` CLI authenticated to GitHub (`gh auth login`)
- Claude Code with MCP Linear server connected (for `/self-code-review` Linear ticket context — optional, degrades gracefully)
