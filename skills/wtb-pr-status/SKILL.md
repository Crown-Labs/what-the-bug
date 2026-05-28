---
name: wtb-pr-status
description: "Snapshot of all open PRs — grouped by review status with CI check results. Run from any git repo."
user-invocable: true
---

# /wtb-pr-status

Snapshot of all open pull requests grouped by status.

## Invocation

```
/wtb-pr-status
```

Run from within any git repository with a GitHub remote.

---

## Step 0 — Foundation

Read and execute the shared foundation before proceeding:
1. Read `shared/foundation.md` (relative to this plugin's root directory)
2. Execute sections A–B as described (C and D not needed for this skill)
3. Store: `REPO_FULL`

---

## Step 1 — Fetch Open PRs

```bash
gh pr list --repo "$REPO_FULL" --state open \
  --json number,title,author,isDraft,reviewDecision,headRefName,createdAt,reviewRequests,assignees
```

Store all open PRs for grouping.

---

## Step 2 — Group by Status

Categorize each PR:

| Condition | Group |
|-----------|-------|
| `isDraft` = true | Draft |
| `reviewDecision` = "APPROVED" | Approved (ready to merge) |
| `reviewDecision` = "CHANGES_REQUESTED" | Changes Requested |
| `reviewDecision` = "" or null, `isDraft` = false | Ready for Review (awaiting first review) |

---

## Step 3 — CI Status

For each open PR, check CI status:

```bash
gh pr checks $PR_NUMBER --repo "$REPO_FULL" 2>/dev/null
```

Categorize:
- All checks pass → `CI ✅`
- Some checks failing → `CI ❌` + name of failing check
- Checks pending → `CI ⏳`
- No checks configured → omit CI status

---

## Step 4 — Review Status

For PRs awaiting review:
- List who has been requested to review (from `reviewRequests`)
- Calculate days since review was requested (from PR `createdAt` or latest review request)

Show assignee if set.

---

## Output

```
🔀 PR Status

✅ Ready to Merge (approved + CI pass)
- PR #50 feat: billing page — @dev2 — approved by @dev1, CI ✅

👀 Awaiting Review
- PR #48 fix: auth timeout — @dev1 — review requested from @dev2 (2 days)

🔧 Changes Requested
- PR #45 feat: rate limit — @dev2 — @dev1 requested changes May 27

📝 Draft
- PR #52 wip: onboarding redesign — @dev2

❌ CI Failing
- PR #48 — test_auth_flow failed
```

**Rules:**
- Omit empty groups
- A PR can appear in both its review group AND the CI Failing section
- Show author for all PRs: `— @username`
- Show assignee if different from author
- Show requested reviewers and days waiting for "Awaiting Review" group
