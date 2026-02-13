---
name: fix-issue
description: Fetch a GitHub issue by number, create a branch off latest main, research the codebase, plan the fix, implement with full test coverage and documentation.
disable-model-invocation: true
allowed-tools: Bash(gh *), Bash(git *), Bash(cargo *), Read, Edit, Write, Grep, Glob
argument-hint: "<issue-number or github-issue-url>"
---

# Fix GitHub Issue

## Step 1: Resolve the issue

Parse `$ARGUMENTS` to extract the issue number:
- If it's a URL like `https://github.com/owner/repo/issues/42`, extract `42`.
- If it's a bare number, use it directly.
- If empty, stop and ask the user for an issue number.

Fetch the issue:

```
gh issue view {number} --json title,body,labels,assignees,comments,state
```

If the issue is closed, warn the user and ask if they still want to proceed.

## Step 2: Create a branch

Create a fresh branch off the latest main (or master):

1. Fetch latest: `git fetch origin`
2. Detect default branch: try `origin/main`, fall back to `origin/master`
3. Create and switch to a new branch: `git checkout -b fix/{number}-{short-slug} origin/{default-branch}`
   - `{short-slug}` is 3-5 words from the issue title, lowercase, hyphenated (e.g. `fix/42-idor-workspace-check`)

If the working tree has uncommitted changes, warn the user and stop. Do not stash or discard their work.

## Step 3: Understand the issue

Summarize the issue in 2-3 sentences. Identify:
- **What's broken or missing** (the symptom or feature request)
- **Acceptance criteria** (what "done" looks like, from the issue body or comments)
- **Constraints** (mentioned technologies, backward compatibility, performance requirements)

If the issue is unclear or ambiguous, list the open questions. These will be addressed during planning.

## Step 4: Research the codebase

Before planning, gather context:

1. **Find relevant code** - Search for files, functions, types, and patterns mentioned in the issue. Read them in full.
2. **Trace the flow** - If the issue is about a specific behavior, trace the code path from the entry point (route handler, CLI command, etc.) through to the relevant logic.
3. **Check existing tests** - Find tests related to the affected code. Understand what's already covered.
4. **Check for prior art** - Look for similar patterns in the codebase that solve analogous problems. Prefer consistency with existing patterns.

## Step 5: Enter planning mode

Enter planning mode to design the implementation. The plan MUST cover:

1. **Root cause** (for bugs) or **design approach** (for features)
2. **Files to modify** with specific descriptions of what changes in each
3. **New files** (if any) with justification for why they're needed
4. **Tests to add** - every code path introduced or changed needs a test:
   - Happy path (expected input produces expected output)
   - Error paths (invalid input, missing data, permission denied)
   - Edge cases (empty collections, boundary values, concurrent access)
   - If the project has e2e tests, include an e2e test for the full flow
5. **Documentation updates**:
   - Module-level doc comments for new or significantly changed modules
   - Doc comments on public functions explaining behavior, parameters, and errors
   - Update architecture docs (e.g. `docs/`, `CLAUDE.md`) if the change adds new concepts, flows, or assumptions
   - Inline comments for non-obvious logic, security invariants, or "why" decisions
6. **Migration or compatibility concerns** (if any)

Follow the project's CLAUDE.md guidance for architecture decisions.

Wait for user approval before implementing.

## Step 6: Implement

After the plan is approved:

1. Implement each change from the plan.
2. Write all planned tests.
3. Update all planned documentation.
4. Run the project's full build/lint/test cycle to verify:
   - Build passes
   - Linter passes with zero warnings
   - Formatter passes
   - All tests pass (both existing and new)
5. If any check fails, fix it before proceeding.

## Step 7: Commit and summarize

1. Commit with a descriptive message referencing the issue (e.g. `fix: prevent IDOR in function call outputs (#42)`).
2. Summarize what was done:
   - Files changed with line references
   - Tests added and what they cover
   - Documentation updated
   - Any follow-up work or open questions
