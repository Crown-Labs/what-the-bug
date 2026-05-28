---
name: wtb-weekly-report
description: "End-of-week summary — throughput, completed work, carry-over, blockers, contributor activity. Run on Fridays from any git repo."
user-invocable: true
---

# /wtb-weekly-report

End-of-week summary for your GitHub project. Best run on Fridays.

## Invocation

```
/wtb-weekly-report
```

Run from within any git repository with a GitHub remote.

---

## Step 0 — Foundation

Read and execute the shared foundation before proceeding:
1. Read `shared/foundation.md` (relative to this plugin's root directory)
2. Execute sections A–D as described
3. Store: `REPO_FULL`, `PROJECT_ID` (if detected), `WEEK_START`, `WEEK_END`

---

## Step 1 — Completed This Week

Fetch issues closed and PRs merged between Monday and Friday of the current week.

```bash
# Closed issues this week
gh issue list --repo "$REPO_FULL" --state closed \
  --search "closed:>=$WEEK_START closed:<=$WEEK_END" \
  --json number,title,assignees,labels

# Merged PRs this week
gh pr list --repo "$REPO_FULL" --state merged \
  --search "merged:>=$WEEK_START merged:<=$WEEK_END" \
  --json number,title,author
```

Group closed issues by Priority (if Projects detected). Show assignee for each.

---

## Step 2 — Opened This Week

```bash
gh issue list --repo "$REPO_FULL" --state all \
  --search "created:>=$WEEK_START created:<=$WEEK_END" \
  --json number \
  --jq 'length'
```

Show count of new issues opened this week.

---

## Step 3 — Throughput

Calculate:
- `CLOSED_COUNT` = number of issues closed this week (from Step 1)
- `OPENED_COUNT` = number of issues opened this week (from Step 2)
- `MERGED_COUNT` = number of PRs merged this week (from Step 1)
- `NET_RESOLVED` = CLOSED_COUNT - OPENED_COUNT

---

## Step 4 — Carry-over

Items that were in progress but not closed by end of week.

**If Projects board detected:**
Query items where Status = "In Progress" and issue state is still open.

**Fallback:**
Open issues that have an assignee and were active this week (had comments or label changes).

```bash
gh issue list --repo "$REPO_FULL" --state open \
  --json number,title,assignees \
  --jq '.[] | select(.assignees | length > 0)'
```

Show assignee and brief status for each.

---

## Step 5 — Blockers This Week

```bash
gh issue list --repo "$REPO_FULL" --state all --label "blocked" \
  --search "updated:>=$WEEK_START" \
  --json number,title,assignees,comments
```

Show issues that had the `blocked` label at any point during the week. If possible, estimate duration blocked from comment timestamps.

---

## Step 6 — Contributor Activity

Aggregate from Steps 1 data:
- Per contributor: count of issues closed, PRs merged
- Show assignee

```bash
# Extract unique contributors from closed issues and merged PRs
# Build a per-contributor summary
```

Parse the JSON from Step 1 to count per-assignee/author.

---

## Output

```
📊 Weekly Report — [MON_DATE]–[FRI_DATE]

📈 Throughput
- Closed: X issues, Y PRs merged
- Opened: Z new issues
- Net: +N resolved

✅ Completed
- #123 Fix login timeout [High/S] — @dev1
- #456 Add billing page [Medium/L] — @dev2

🔄 Carry-over (still in progress)
- #789 Refactor auth — @dev1 — continues next week
- #101 Payment integration — @dev2 — blocked mid-week, unblocked Thu

🚧 Blockers This Week
- #202 waited 3 days on API key — @dev1

👥 Contributor Activity
- @dev1: 4 issues closed, 3 PRs merged
- @dev2: 2 issues closed, 2 PRs merged
```

**Rules:**
- Omit empty sections
- Show Priority + Size: `[High/S]` for completed items (if Projects detected)
- Show assignee for all items
- Date range: `May 26–May 30, 2026`
