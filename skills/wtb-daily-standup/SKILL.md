---
name: wtb-daily-standup
description: "Morning standup report — closed yesterday, in progress today, blocked items, and candidates to pick up. Run from any git repo."
user-invocable: true
---

# /wtb-daily-standup

Morning standup report for your GitHub project.

## Invocation

```
/wtb-daily-standup
```

Run from within any git repository with a GitHub remote.

---

## Step 0 — Foundation

Read and execute the shared foundation before proceeding:
1. Read `shared/foundation.md` (relative to this plugin's root directory)
2. Execute sections A–D as described
3. Store: `REPO_OWNER`, `REPO_NAME`, `REPO_FULL`, `PROJECT_ID` (if detected), date helpers

---

## Step 1 — Done Yesterday

Fetch issues closed and PRs merged since yesterday. Use the `YESTERDAY` variable from Step 0 (skips weekends automatically).

```bash
# Closed issues since yesterday
gh issue list --repo "$REPO_FULL" --state closed \
  --search "closed:>=$YESTERDAY" \
  --json number,title,assignees \
  --jq '.[] | "#\(.number) \(.title) — \(.assignees | map("@" + .login) | join(", ") | if . == "" then "unassigned" else . end)"'

# Merged PRs since yesterday
gh pr list --repo "$REPO_FULL" --state merged \
  --search "merged:>=$YESTERDAY" \
  --json number,title,author \
  --jq '.[] | "PR #\(.number) \(.title) — @\(.author.login)"'
```

Combine results into the "Done Yesterday" section.

---

## Step 2 — In Progress Today

**Primary (if `PROJECT_ID` is set):**

Query the Projects board for items with Status = "In Progress":

```bash
gh api graphql -f query='
  query($projectId: ID!) {
    node(id: $projectId) {
      ... on ProjectV2 {
        items(first: 50) {
          nodes {
            fieldValueByName(name: "Status") {
              ... on ProjectV2ItemFieldSingleSelectValue { name }
            }
            content {
              ... on Issue {
                number
                title
                assignees(first: 5) { nodes { login } }
              }
            }
          }
        }
      }
    }
  }
' -f projectId="$PROJECT_ID" 2>/dev/null
```

Filter items where Status field value = "In Progress". Show issue number, title, and assignees.

**Fallback (no Projects board):**

```bash
gh issue list --repo "$REPO_FULL" --state open \
  --json number,title,assignees \
  --jq '.[] | select(.assignees | length > 0)'
```

Show assigned open issues as the "in progress" proxy.

---

## Step 3 — Blocked Items

```bash
gh issue list --repo "$REPO_FULL" --state open --label "blocked" \
  --json number,title,assignees,comments \
  --jq '.[] | {number, title, assignees: (.assignees | map(.login)), lastComment: (.comments | last | .body // "no comment")}'
```

For each blocked issue:
- Show issue number, title, assignee
- Extract the most recent comment for the blocked reason (first 100 chars)
- Show how long it's been blocked if determinable from comment date

---

## Step 4 — Candidates to Pick Up

```bash
gh issue list --repo "$REPO_FULL" --state open \
  --json number,title,assignees,labels
```

Filter: issues with empty `assignees` array (unassigned).
Exclude: issues with label `blocked`.

**If Projects board detected:**
Query Priority and Size fields for each candidate issue, then sort:
1. Priority: High → Medium → Low → unset
2. Within same priority: Size S → M → L → XL → unset (smaller first)

**If no Projects board:**
Show in creation order (newest first).

Display top 5 candidates with `[Priority/Size]` prefix.

---

## Output

Format the combined results using this template:

```
📅 Daily Standup — [DATE]

✅ Done Yesterday
- #123 Fix login timeout — @dev1 (merged)
- #456 Update API docs — @dev2 (closed)

🔄 In Progress Today
- #789 Add billing page — @dev2
- #101 Refactor auth — @dev1

🚧 Blocked
- #202 Payment gateway — @dev1 — waiting on 3rd party API key

📋 Candidates (unassigned, ready to pick up)
- #303 [High/S] Add rate limiting
- #404 [Medium/M] Improve error messages
- #505 [Medium/S] Update onboarding flow
```

**Rules:**
- If no items in a section → omit that section entirely
- Show assignee: `— @username` or `— unassigned`
- Show Priority + Size: `[High/S]` for candidates
- Date format: `May 28, 2026`
- No markdown tables — bullet lists only
