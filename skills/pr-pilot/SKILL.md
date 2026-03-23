---
name: pr-pilot
description: Manages a single GitHub pull request through its lifecycle — triages what needs attention, implements fixes, responds to reviewers, rebases, and gets the PR to merge. This skill MUST be used whenever the user's request involves an existing pull request that needs maintenance, attention, or action. Trigger on any of these patterns - mentioning a PR number or URL ("PR #42", "pull/89"), asking about PR status or blockers ("what's blocking my PR", "is my PR ready to merge", "check on my PR"), asking to handle review feedback ("address the review comments", "handle the feedback", "implement the suggestions", "apply the reviewer's changes"), asking to fix PR issues ("CI is failing on my PR", "my PR has merge conflicts", "PR needs rebasing", "rebase my branch"), asking to get a PR merged ("get it ready to merge", "get this landed", "merge my PR"), or referring to reviewer interactions on an existing PR ("reviewer asked for changes", "apply the suggestions", "respond to the review"). Do NOT trigger for creating new PRs, doing code reviews, or general git operations without a PR context.
---

# PR Pilot

A workflow for triaging and maintaining a single GitHub pull request — from review comments through to merge.

## Prerequisites

- `gh` CLI authenticated and available
- Current directory is within the repository (or user provides owner/repo context)

## Step 1: Identify the PR

Determine which PR to work on, in this order:

1. **User provided a PR number or URL** — extract the number directly
2. **Current branch has an open PR** — run `gh pr view --json number,url,title` on the current branch
3. **Ask the user** — if neither works, ask which PR they want to manage

Once identified, confirm briefly: "Working on PR #N: <title>"

## Step 2: Gather full PR state

Fetch everything in parallel using `gh`:

```bash
# Core PR details
gh pr view <N> --json title,state,baseRefName,headRefName,mergeable,mergeStateStatus,reviewDecision,statusCheckRollup,labels,assignees,reviewRequests,url,body,isDraft

# Review threads (comments organized by conversation)
gh api repos/{owner}/{repo}/pulls/{N}/comments --paginate

# Top-level PR review summaries (APPROVED, CHANGES_REQUESTED, etc.)
gh api repos/{owner}/{repo}/pulls/{N}/reviews --paginate

# CI/check details (if statusCheckRollup needs more info)
gh pr checks <N>
```

Also check if the branch is behind the base branch:

```bash
git fetch origin
git rev-list --left-right --count origin/<base>...origin/<head>
```

**Check the PR body/description** for outstanding items — look for unchecked checkboxes (`- [ ]`), TODO items, or open questions the author left for themselves or reviewers. These often contain prerequisites or follow-up items that affect merge readiness. The `body` field from `gh pr view --json body` contains this.

## Step 3: Triage and summarize

Present a concise summary organized by what needs attention. Use this priority order:

### Blocking items (fix these first)
- **Merge conflicts** — the branch cannot be merged as-is
- **CI failures** — checks are failing
- **Changes requested** — reviewers have requested changes

### Actionable items
- **Unresolved review comments** — comments that haven't been addressed or replied to
- **Branch is behind base** — needs rebase (even without conflicts, this can cause CI issues)
- **Unchecked PR body items** — outstanding TODO checkboxes or open questions in the PR description
- **Reviewer approval gaps** — reviewers who left feedback (especially change requests) but haven't posted a formal approval after their feedback was addressed. This matters because branch protection rules may require their approval, and it signals the reviewer hasn't verified their concerns were resolved

### Status
- **Approvals received** — which reviewers have approved
- **Checks passing** — which CI checks are green
- **Draft status** — whether the PR is still marked as draft

After the summary, state what you plan to do and in what order. Then proceed unless the user intervenes.

## Step 4: Set up worktree and work through items

All modifications happen in an isolated git worktree. This allows concurrent agents to work on the same repository without interfering with each other.

### Create the worktree

```bash
# Generate a unique worktree path to avoid clashing with other sessions on the same PR
WORKTREE_DIR="/tmp/pr-<N>-worktree-$(date +%s)-$$"
git worktree add "$WORKTREE_DIR" origin/<head-branch>
cd "$WORKTREE_DIR"

# Ensure you're on the correct branch (not detached HEAD)
git checkout <head-branch>
```

All subsequent file modifications, commits, and pushes happen inside this worktree. Do not modify files in the original working directory.

Now process items in priority order. The general principle: investigate thoroughly before acting, and pause for decisions that require judgment.

### 4a. Review comments — investigate before acting

The goal is to provide well-considered recommendations, not to blindly implement suggestions. Reviewer feedback is input to evaluate, not instructions to follow.

#### Gather context first

```bash
gh pr diff <N>
```

Read the full diff plus any files referenced by review comments. Understand the broader context — why does this code exist, what does it connect to, what invariants does it maintain?

#### Investigate each comment thread

For every unresolved review comment, perform this investigation before deciding what to do:

1. **Understand the concern** — restate the reviewer's point in your own words. What problem are they identifying? What change are they proposing?

2. **Verify against the codebase** — check whether the reviewer's observation is technically correct for this codebase. Read surrounding code, check call sites, look at tests, and trace data flow. Reviewers sometimes lack full context or misread the diff.

3. **Evaluate the suggestion** — would implementing it actually improve things? Consider:
   - Does the current implementation exist for a reason the reviewer may have missed?
   - Would the suggested change break existing functionality or tests?
   - Does it conflict with patterns established elsewhere in the codebase?
   - Is it solving a real problem or a hypothetical one (YAGNI)?

4. **Consider alternatives** — if the reviewer identified a real issue but the proposed fix isn't ideal, think about what a better solution would look like.

#### Present findings and recommendations

After investigating all comment threads, present a structured summary to the user:

For each comment, include:
- **Reviewer's concern**: one-line summary
- **Investigation finding**: what you found when you checked the codebase
- **Recommendation**: one of:
  - **Agree and implement** — the suggestion is correct and improves the code. Note what you'll change.
  - **Partially agree** — the reviewer found a real issue but the proposed fix needs adjustment. Explain what you'd do instead and why.
  - **Disagree** — the suggestion is technically incorrect, breaks things, or isn't needed. Provide specific evidence (test results, call sites, existing patterns).
  - **Need clarification** — the comment is ambiguous or you can't determine the right course without more context.

Wait for the user to confirm or adjust the plan before implementing any changes. This prevents wasted effort on changes the user might not want, and ensures the user understands the tradeoffs.

#### Implement confirmed changes

Once the user approves the plan, implement changes in the worktree. For each change:
- Make the modification
- Run relevant tests if available to verify nothing breaks
- Track which review comments were addressed so you can reply in Step 5

### 4b. CI failures

If checks are failing:

1. Get the failure details: `gh pr checks <N>` and follow up with `gh run view <run-id> --log-failed` for the specific failing run
2. Read the failure logs to understand what went wrong
3. If the fix is straightforward (test failure due to a change you just made, linting issue, type error), fix it
4. If the failure is flaky or infrastructure-related, tell the user and suggest re-running: `gh run rerun <run-id> --failed`
5. If the failure is complex, explain what you found and discuss with the user

### 4c. Rebase

If the branch is behind the base branch or has merge conflicts:

1. Make sure the working tree is clean (stash or commit pending changes first)
2. Fetch latest and rebase:
   ```bash
   git fetch origin
   git rebase origin/<base>
   ```
3. **If conflicts arise**: attempt to resolve them intelligently by reading both sides, understanding the intent, and choosing the correct resolution. After resolving, present a summary of what you resolved and how — give the user a chance to review before continuing
4. After successful rebase, force-push (with lease for safety):
   ```bash
   git push --force-with-lease origin <head-branch>
   ```

Always use `--force-with-lease` rather than `--force` to avoid overwriting someone else's pushes.

## Step 5: Commit, push, and reply

After implementing fixes:

1. **Commit changes** with a clear message referencing the review feedback:
   ```
   Address review feedback on PR #N

   - Fix <what was fixed>
   - Update <what was updated>
   ```

2. **Push** to the PR branch:
   ```bash
   git push origin <head-branch>
   ```

3. **Reply to addressed comments** — for each review comment you fixed, post a brief reply indicating what you did:
   ```bash
   gh api repos/{owner}/{repo}/pulls/{N}/comments/{comment_id}/replies \
     -f body="Done — <brief description of what was changed>"
   ```

   Keep replies concise. Reviewers appreciate knowing their feedback was addressed without having to re-read the diff themselves.

4. **Resolve conversations** where appropriate — if a comment was fully addressed, the reply itself signals this. Don't mark conversations as resolved on behalf of the reviewer unless the user asks you to.

## Step 6: Clean up worktree and final status

Return to the original working directory and remove the worktree:

```bash
cd <original-working-directory>
git worktree remove "$WORKTREE_DIR"
```

If the removal fails (e.g., uncommitted changes left behind), force-remove it:
```bash
git worktree remove --force "$WORKTREE_DIR"
```

Present an updated summary:

- What was changed and committed
- Which review comments were addressed (and which were left for the user)
- Current CI status (may need to wait for checks to start)
- Whether the PR is now mergeable

If everything looks good and the PR has sufficient approvals, ask the user if they'd like to merge it. Recommend a merge strategy based on the PR:

- **Squash merge** — for small/docs PRs, single logical changes, or PRs with noisy commit history (merge commits, fixup commits)
- **Merge commit** — for PRs with multiple distinct, well-structured commits that are worth preserving individually
- **Rebase merge** — for PRs with a clean linear commit history that should stay linear on main

If a reviewer left feedback that was addressed but hasn't formally re-approved, flag this — the user may want to ping them before merging, especially if branch protection requires their approval.

## Important guidelines

- **Never force-push without `--force-with-lease`** — this protects against overwriting collaborators' commits
- **Don't resolve review conversations yourself** — let the reviewer do that, unless the user explicitly asks
- **Don't merge without asking** — always confirm with the user before merging
- **Don't dismiss reviews** — even if you've addressed all feedback, leave review dismissal to the user
- **Keep the user informed** — when you auto-fix something, briefly note what you did so they can verify
- **Respect branch protection rules** — if the repo requires certain checks or approvals, don't try to work around them
- **Always clean up worktrees** — if the run is interrupted or errors out, ensure the worktree is removed before exiting. Leaked worktrees waste disk space and can cause branch conflicts
- **One PR at a time** — this skill manages a single PR per invocation. If the user wants to manage multiple PRs, they should invoke it separately for each one
