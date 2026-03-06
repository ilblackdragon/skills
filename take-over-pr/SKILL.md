---
name: take-over-pr
description: Take over a stale or abandoned PR. Checks out the work, merges main, deep-reviews the code, addresses all review comments, then re-creates a clean PR crediting the original author.
disable-model-invocation: true
allowed-tools: Bash(gh *), Bash(git *), Bash(cargo *), Bash(just *), Bash(make *), Bash(npm *), Bash(pnpm *), Read, Edit, Write, Grep, Glob, Task
argument-hint: "<pr-number or github-pr-url>"
---

# Take Over Pull Request

Take ownership of an existing PR that is stale, abandoned, or needs significant work to land. The goal: get the work merged into main as fast as possible while crediting the original author.

## Step 1: Resolve the PR

Parse `$ARGUMENTS` to extract the PR number:
- URL like `https://github.com/owner/repo/pull/123` -> extract `123`.
- Bare number -> use directly.
- Empty -> stop and ask the user for a PR number.

Fetch full PR metadata:

```
gh pr view {number} --json number,title,body,author,baseRefName,headRefName,labels,state,additions,deletions,files,commits,reviewRequests,statusCheckRollup
```

If the PR is already merged, tell the user and stop.

Record: `{number}`, `{owner}`, `{repo}`, `{author_login}`, `{title}`, `{body}`, `{head_branch}`, `{base_branch}`.

## Step 2: Gather all review activity

Fetch everything reviewers said:

```
gh api repos/{owner}/{repo}/pulls/{number}/comments
gh api repos/{owner}/{repo}/pulls/{number}/reviews
gh api repos/{owner}/{repo}/issues/{number}/comments
```

Compile a deduplicated list of review findings. Group by the actual issue, not by comment ID (bots and humans sometimes raise the same thing multiple times).

Classify each as:
- **Unresolved** - needs a code change or reply
- **Resolved** - author already addressed it in a later commit
- **Informational** - bot status, CI report, no action needed

## Step 3: Fetch the full diff and changed file list

```
gh pr diff {number}
gh pr diff {number} --name-only
```

## Step 4: Check out the work onto a fresh branch

Ensure working tree is clean. If there are uncommitted changes, warn and stop.

```
git fetch origin
```

Create a new takeover branch from the latest base branch:

```
git checkout -b takeover/{number}-{short-slug} origin/{base_branch}
```

Where `{short-slug}` is 3-5 words from the PR title, lowercase, hyphenated.

Now merge the PR's head branch to bring in the original author's work:

```
git fetch origin pull/{number}/head:pr-{number}-incoming
git merge pr-{number}-incoming --no-edit
```

If there are merge conflicts:
1. List every conflicted file.
2. For each conflict, read both sides and the original PR diff to understand the author's intent.
3. Resolve conflicts preserving the author's intent while adapting to current main.
4. Stage resolved files.
5. Complete the merge commit.

Delete the temporary ref:

```
git branch -D pr-{number}-incoming
```

## Step 5: Deep code review

Now perform a thorough review of the merged changes. This is the same rigor as `/review-code`:

```
git diff origin/{base_branch}...HEAD --name-only
git diff origin/{base_branch}...HEAD
```

For each changed file, read the ENTIRE file (not just diff hunks) to understand:
- What the change does in full context
- Whether it follows project patterns (check CLAUDE.md)
- Whether it introduces bugs, security issues, or missing edge cases

### Review lenses

Apply each of these:

**Correctness**: Off-by-one, wrong comparisons, unreachable code, broken invariants, concurrency issues, incorrect error propagation.

**Edge cases**: Empty input, None/null, zero-length collections, integer overflow, malformed input, partial failures.

**Security**: Auth bypass, injection, data leakage, resource exhaustion, replay attacks, timing attacks.

**Tests**: Is every new public function tested? Error paths? Edge cases? Do existing tests still make sense?

**Project conventions**: Does the code follow CLAUDE.md? Correct import style? Error handling pattern? No unwraps in non-test code?

### Compare against review comments

Cross-reference your findings with the review comments from Step 2. For each reviewer comment:
1. Was the issue real? Read the code to verify.
2. Is it already fixed by a later commit in the PR? Check the commit history.
3. If still open, add it to your fix list.

## Step 6: Plan and present

Compile all issues into a single plan, combining:
- Unresolved review comments (from Step 2)
- New issues found during deep review (from Step 5)
- Merge conflict follow-ups (from Step 4, if any)

Present to the user as a table:

| # | Source | Severity | File:Line | Issue | Planned Fix |
|---|--------|----------|-----------|-------|-------------|

Source is one of: `review-comment`, `new-finding`, `merge-followup`.

Severity levels: Critical, High, Medium, Low, Nit.

**Wait for user confirmation before implementing fixes.**

## Step 7: Implement fixes

After user confirms:

1. Fix each issue from the plan.
2. Run the project's full build/lint/test cycle:
   - Check for `justfile` targets first.
   - Otherwise: `cargo fmt && cargo clippy --all --benches --tests --examples --all-features && cargo test`
   - For non-Rust projects, use the appropriate toolchain.
3. If any check fails, fix it before proceeding.
4. Commit the fixes separately from the merge. Use a descriptive message:
   ```
   fix: address review feedback and code improvements (takeover #{number})
   ```

## Step 8: Push and create new PR

Push the takeover branch:

```
git push -u origin takeover/{number}-{short-slug}
```

Create a new PR. The description MUST:
- Link to the original PR
- Credit the original author
- Summarize what the original PR did
- List what was fixed/improved during takeover
- Include Co-Authored-By in the PR body

Use this template:

```
gh pr create --title "{improved title if needed, otherwise original title}" --body "$(cat <<'EOF'
## Summary

Continuation of #{number} by @{author_login}.

{Original PR summary, rewritten concisely in your own words}

## Changes from original

- Merged with latest {base_branch} (resolved N conflicts)
- {List each fix/improvement made during takeover}

## Original PR

#{number} - {original_title}

## Review comments addressed

{For each resolved comment: one-line summary of the issue and fix}

## Test plan

- [ ] All existing tests pass
- [ ] {Any new tests added}
- [ ] CI green

---

Co-Authored-By: {author_login} <{author_login}@users.noreply.github.com>

Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

## Step 9: Comment on the original PR

Leave a polite comment on the original PR linking to the new one:

```
gh pr comment {number} --body "$(cat <<'EOF'
Thanks for the work on this, @{author_login}! I've picked up your changes and continued them in #{new_pr_number}.

The new PR includes your original work plus:
- Merge with latest {base_branch}
- Fixes for review feedback
- {Any other improvements}

You're credited as co-author on all commits. Feel free to review the new PR!
EOF
)"
```

Do NOT close the original PR automatically. Ask the user if they want to close it.

## Step 10: Report

Summarize the full takeover:

| Step | Status | Details |
|------|--------|---------|
| Original PR | #{number} by @{author_login} | {title} |
| New PR | #{new_number} | {new_url} |
| Merge conflicts | N resolved | {files if any} |
| Review comments addressed | N | {summary} |
| New issues found and fixed | N | {summary} |
| Build/lint/test | PASS/FAIL | |

If anything remains unresolved, list it clearly.

## Rules

- **Credit is sacred.** Always credit the original author with Co-Authored-By and explicit mentions. They did the hard initial work.
- Never force-push. Always regular push.
- Never close the original PR without asking the user first.
- Read every file before editing it. No blind fixes.
- Run lint/test locally before every push. Don't push broken code.
- Stay focused on landing the PR. Don't refactor unrelated code.
- If merge conflicts are too complex to resolve confidently, stop and ask the user.
- If review comments are ambiguous or you disagree with them, present your reasoning to the user rather than silently ignoring.
- Prefer the original author's approach when it's reasonable. You're here to polish and land their work, not rewrite it.
