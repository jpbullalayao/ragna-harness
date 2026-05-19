---
name: self-code-review
description: Review the current branch's diff against the latest origin/main. Flags functionality regressions, unnecessary uncommon hooks (useRef/useEffect/useMemo/useCallback), duplicated helpers or components that already exist in the codebase, and stale leftover code from iteration. Use when the user runs /self-code-review or asks for a review of their current branch, a pre-PR check, or "review my changes".
allowed-tools:
  - "Bash(git fetch *)"
  - "Bash(git log *)"
  - "Bash(git diff *)"
  - "Bash(git status *)"
  - "Bash(git rev-parse *)"
  - "Bash(git merge-base *)"
  - "Bash(git show *)"
  - "Bash(git branch *)"
---

# Code Review

Reviews the current branch against the latest `origin/main`. **Read-only** — never mutates the working tree (no checkout, pull, commit, push, or stash).

## Workflow

### Step 1: Establish the diff baseline

Run these in parallel:

```bash
git rev-parse --abbrev-ref HEAD
git fetch origin main --quiet
```

- If the current branch is `main` or `master`, stop and tell the user there is nothing to review (they need to be on a feature branch).
- If `git fetch` fails (no remote, offline, etc.), fall back to local `main` and note the limitation in the output.

Then compute the merge base:

```bash
git merge-base origin/main HEAD
```

Capture the resulting SHA — call it `BASE` for the rest of the workflow.

### Step 2: Resolve Linear ticket context

The Summary should explain the *problem* this PR is meant to solve, not just the *what* of the diff. Pull that context from Linear when available.

1. Extract candidate ticket IDs from:
   - The current branch name (e.g. `kor-1449-ulr-into-ibnr` → `KOR-1449`).
   - Commit subjects from `git log <BASE>..HEAD --oneline` (e.g. `feat(bdx): ... (KOR-1449)`, `fix(ui): ... (KOR-1385/1449)`).
   - Use the regex `[A-Za-z][A-Za-z0-9]+-\d+`, upper-case the matches, and dedupe.

2. If one or more IDs are found, invoke the **`linear`** skill and call `mcp__linear-server__get_issue` for each unique ID. Capture `title`, `description`, `state`, `priority`, and `url`.
   - Cap at **3 ticket fetches** per review. If more IDs are referenced, fetch the first three (branch-name match first, then commits in order) and note the rest in **Other Notes**.
   - Tolerate failures: if MCP isn't connected, the ticket is archived, or permissions block the read, log "Could not fetch Linear ticket KOR-XXXX — falling back to commit/diff inference" and continue. Never block the review on a Linear failure.

3. If no IDs are found at all, skip the fetch and surface that in the **Problem** section of the output (see Step 5).

### Step 3: Gather diff context

Run these in parallel (substitute the captured `BASE`):

```bash
git log <BASE>..HEAD --oneline
git diff --stat <BASE>..HEAD
git diff --name-status <BASE>..HEAD
git diff <BASE>..HEAD
```

If the diff is empty, stop and tell the user the branch has no changes vs `origin/main`.

**For large diffs** (> ~30 files or > ~1500 lines): don't rely on diff hunks alone. Read the full contents of the most-changed files so you can spot code that was *removed but is still referenced elsewhere*, and dispatch an Explore subagent to search for cross-file impacts (callers of removed/renamed symbols, similar utilities elsewhere in the repo).

### Step 4: Apply the four review guidelines

#### a) Functionality regressions
Did this branch accidentally remove or break something that worked before?

- Removed exports, props, event handlers, or branches in `if`/`switch`.
- Narrowed conditionals (e.g. `if (a || b)` → `if (a)`) without justification.
- Signature changes (added required params, changed return shape) — grep callers to confirm they were updated.
- Renamed/moved files — grep for old import paths.
- Removed cases from a discriminated union or enum without updating exhaustive switches.
- Behavior changes in shared utilities that callers depend on.

#### b) Unnecessary uncommon hooks
Scan additions for `useRef`, `useEffect`, `useMemo`, `useCallback`, `useImperativeHandle`, `useLayoutEffect`.

For each new occurrence, judge whether it's load-bearing:

- **Likely unnecessary**: `useEffect` that syncs derived state (compute it during render); `useMemo`/`useCallback` whose dep array changes every render anyway; `useRef` storing values that could be props or local variables; `useEffect` for fetching data in a Server Component or that should be an event handler.
- **Legitimate**: DOM measurement / focus, integrating non-React libraries, subscribing to external stores, debounce/throttle timers, stable identity required by a downstream `memo` boundary.

When flagging, suggest the simpler alternative (derived state, event handler, server component, ref-as-prop, etc.). Don't flag hooks that are clearly load-bearing — balance code quality against churn.

#### c) Duplicated helpers / brand components
For each *new* utility function, hook, or component in the diff:

- Grep the repo for similar names (`<name>`, partial matches, common synonyms).
- For components, look in shared UI packages (`packages/ui`, `packages/design-system`, `apps/*/components`, `apps/*/lib`, etc.).
- For utilities, look in `packages/utils`, `packages/core`, `apps/*/lib`, `lib/`, `utils/`.
- Use Glob + Grep, or dispatch an Explore subagent if the surface area is large.

When you find a near-duplicate, cite the existing path and suggest importing from there instead.

#### d) Stale code from iteration
Anything left behind that shouldn't ship:

- `console.log`, `console.debug`, `debugger`, `alert`.
- Commented-out code blocks.
- Unused imports (in the diff context).
- New exports nothing imports.
- Leftover `TODO`, `FIXME`, `WIP`, `XXX` markers tied to this change.
- Mock data, fake delays, or hardcoded test values in production paths.
- Feature flag / experiment scaffolding that's no longer toggled.
- Half-finished function bodies, dead branches behind `if (false)`, etc.

Skip nits Biome / Ultracite / ESLint already catch — focus on substance.

### Step 5: Output

Use this exact structure. Keep it concise — every finding must cite `file:line` so the reviewer can jump directly. Use "None spotted" rather than omitting a section, so the reviewer can see each guideline was considered.

```markdown
## Problem
<2–4 sentences pulled from the Linear ticket(s) fetched in Step 2: the user-facing problem, who reported it / what triggered it, and the acceptance criteria or "done" condition stated in the ticket. Cite each ticket as a markdown link in `[KOR-1449](url)` form.

If no Linear ID was found in the branch or commits, write: "No Linear ticket linked from branch or commits — Problem inferred from commit messages: <one-line inference>."

If a ticket was referenced but the fetch failed, write: "Linear ticket KOR-XXXX referenced but could not be fetched (<reason>) — Problem inferred from commit messages: <one-line inference>.">

## Summary
<1–3 sentences: what the PR actually changes, and whether that matches the Problem above. If the implementation goes beyond the ticket (or stops short of it), say so here — that's the load-bearing observation. Don't restate the diff line-by-line.>

## Review Order
<Include this section when the diff is large (>10 files or >500 lines) OR touches any DB-migration-related files (`packages/db/drizzle/*.sql`, `packages/db/src/schema/**`, `packages/db/drizzle/meta/*`, or `packages/db/src/scripts/backfill-*`). Otherwise skip.

Before grouping, collect the **complete file list** from the `git diff --name-status <BASE>..HEAD` output gathered in Step 3 — this is the authoritative set every group must draw from.

When DB-migration files are present, list them as the FIRST grouping — schema context is load-bearing for everything downstream. Look for: new columns/types, NOT-NULL adds without defaults, dropped or renamed columns still referenced by app code, index/constraint changes, backfill correctness.

Then list remaining file groupings in dependency order (schema → core/lib → API/server → UI), each with a one-line "what to look for" hint.

**Completeness check**: after forming all groups, verify every file from the complete list appears in exactly one group. Any file with no obvious grouping goes into an "Other" group — never drop a file silently.>

## Findings

### Functionality Regressions
- `path/to/file.ts:42` — <what changed> | Suggest: <fix>
- (or: None spotted.)

### Hook Usage
- `path/to/component.tsx:15` — `useRef` here is unnecessary because <reason>. Suggest: <alternative>.
- (or: None spotted.)

### Duplication / Reuse Opportunities
- `path/to/new-helper.ts` duplicates `existing/path/helper.ts`. Suggest: import from there.
- (or: None spotted.)

### Stale Code
- `path/to/file.ts:88` — leftover `console.log`. Remove.
- (or: None spotted.)

## Other Notes
<Optional. Brief. Things outside the four buckets that genuinely matter: missing tests for a new branch, a11y regression, type-safety gap, security concern. Skip if there's nothing.>
```

## Constraints

- **Read-only.** Never run `git checkout`, `git pull`, `git stash`, `git commit`, `git push`, or anything else that mutates the tree.
- **Be specific.** Every finding must cite a path and (where possible) a line number.
- **Don't restate the diff.** Assume the reviewer can read it. Add value via judgment.
- **Don't flag what the linter catches.** Focus on architecture, regressions, and reuse.
- **Balance.** Some hooks are necessary, some duplication is intentional, some "stale" code is actually intentional scaffolding for a follow-up. When in doubt, frame the finding as a question.
- **Linear context is advisory, not authoritative.** If the ticket and the diff disagree, surface the disagreement as a finding rather than trusting either source blindly. Tickets get stale; PRs sometimes do more (or less) than the ticket says.
