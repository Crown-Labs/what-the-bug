---
name: wtb-blocked-alert
description: "Scan for blocked issues, stale issues, stale PRs, and merge conflicts. Run from any git repo."
user-invocable: true
---

# /wtb-blocked-alert

Scan for blocked items, stale issues, stale PRs, and PRs with merge conflicts.

## Invocation

```
/wtb-blocked-alert
```

Run from within any git repository with a GitHub remote.

---

## Step 0 — Foundation

Read and execute the shared foundation before proceeding:
1. Read `shared/foundation.md` (relative to this plugin's root directory)
2. Execute sections A–D as described
3. Store: `REPO_FULL`, `TODAY`

---

## Step 1 — Blocked Issues

```bash
gh issue list --repo "$REPO_FULL" --state open --label "blocked" \
  --json number,title,assignees,updatedAt,comments
```

For each blocked issue:
- Show issue number, title, assignee
- Calculate days blocked: difference between now and the most recent comment or label-change date
- Show the blocked reason from the latest comment (first 100 chars)

---

## Step 2 — Stale Issues

Open issues with no activity for more than 3 working days.

```bash
# Calculate the cutoff date (3 working days ago)
STALE_CUTOFF=$(TZ=Asia/Bangkok python3 -c "
from datetime import datetime, timedelta
today = datetime.now()
days_back = 0
working_days = 0
while working_days < 3:
    days_back += 1
    d = today - timedelta(days=days_back)
    if d.weekday() < 5:
        working_days += 1
print(d.strftime('%Y-%m-%d'))
")

gh issue list --repo "$REPO_FULL" --state open \
  --json number,title,assignees,updatedAt \
  --jq "[.[] | select(.updatedAt < \"${STALE_CUTOFF}T00:00:00Z\")]"
```

"Activity" includes: comments, label changes, assignee changes, state changes. The `updatedAt` field from GitHub captures all of these.

Show assignee + last update date for each stale issue.

---

## Step 3 — Stale PRs

Open PRs awaiting review for more than 2 working days.

```bash
REVIEW_CUTOFF=$(TZ=Asia/Bangkok python3 -c "
from datetime import datetime, timedelta
today = datetime.now()
days_back = 0
working_days = 0
while working_days < 2:
    days_back += 1
    d = today - timedelta(days=days_back)
    if d.weekday() < 5:
        working_days += 1
print(d.strftime('%Y-%m-%d'))
")

gh pr list --repo "$REPO_FULL" --state open \
  --json number,title,author,reviewRequests,createdAt,updatedAt \
  --jq "[.[] | select(.updatedAt < \"${REVIEW_CUTOFF}T00:00:00Z\")]"
```

Show: PR number, title, author, requested reviewers, days waiting.

---

## Step 4 — PRs with Merge Conflicts

```bash
gh pr list --repo "$REPO_FULL" --state open \
  --json number,title,author,mergeable
```

Filter for PRs where `mergeable` is `"CONFLICTING"`. Show PR number, title, author.

---

## Output

```
🚨 Blocked & Stale Alert

🔴 Blocked Issues
- #202 Payment gateway — @dev1 — blocked 3 days (since May 25)

🟡 Stale Issues (no activity > 3 working days)
- #303 Rate limiting — @dev1 — last update May 23
- #505 Onboarding — unassigned — last update May 22

⏳ PRs Awaiting Review > 2 working days
- PR #45 by @dev2 — review requested from @dev1 — waiting 3 days

💥 PRs with Merge Conflicts
- PR #42 by @dev1 — conflicts with develop
```

If nothing found in any category → output: `✅ All clear — no blocked or stale items`

**Rules:**
- Omit empty sections
- Show assignee for all items
- Working days only (skip weekends in calculations)
- "Days blocked/waiting" = working days, not calendar days
