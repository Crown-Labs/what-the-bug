---
name: wtb-issue-triage
description: "Find untriaged issues and auto-suggest Priority, Size, labels, and milestone. Apply suggestions on user confirm. Run from any git repo."
user-invocable: true
---

# /wtb-issue-triage

Find untriaged issues and auto-classify them. Apply suggestions on user confirm.

## Invocation

```
/wtb-issue-triage
```

Run from within any git repository with a GitHub remote.

---

## Step 0 — Foundation

Read and execute the shared foundation before proceeding:
1. Read `shared/foundation.md` (relative to this plugin's root directory)
2. Execute sections A–D as described (Projects detection needed for Priority/Size fields)
3. Store: `REPO_FULL`, `PROJECT_ID`, field IDs

---

## Step 1 — Find Untriaged Issues

An issue is "untriaged" if it's missing any of: type label, Priority (Projects field), Size (Projects field), or milestone.

```bash
# Fetch all open issues with their labels and milestone
gh issue list --repo "$REPO_FULL" --state open \
  --json number,title,body,assignees,labels,milestone \
  --limit 100
```

**Type labels to check for:** `bug`, `feature`, `enhancement`, `tech-debt`, `research`

Filter issues missing at least one of:
1. No type label (none of the above labels present)
2. No milestone set
3. If Projects detected: no Priority or Size set (check via GraphQL)

If Projects detected, also fetch Projects field values for each issue:

```bash
gh api graphql -f query='
  query($projectId: ID!) {
    node(id: $projectId) {
      ... on ProjectV2 {
        items(first: 100) {
          nodes {
            content {
              ... on Issue { number }
            }
            fieldValueByName(name: "Priority") {
              ... on ProjectV2ItemFieldSingleSelectValue { name }
            }
            fieldValueByName(name: "Size") {
              ... on ProjectV2ItemFieldSingleSelectValue { name }
            }
          }
        }
      }
    }
  }
' -f projectId="$PROJECT_ID" 2>/dev/null
```

Cross-reference: issues without Priority or Size in the Projects board are untriaged.

---

## Step 2 — Analyze Each Issue

For each untriaged issue, read its title and body. Suggest:

**Type label** — based on keywords:
- Contains "bug", "error", "broken", "fix", "crash", "regression" → `bug`
- Contains "add", "new", "feature", "implement" → `feature` or `enhancement`
- Contains "refactor", "cleanup", "tech debt", "improve code" → `tech-debt`
- Contains "investigate", "research", "spike", "explore" → `research`
- Default → `enhancement`

**Priority** — based on severity signals:
- Keywords like "critical", "urgent", "security", "data loss", "production" → High
- Keywords like "important", "should", "needs" → Medium
- Keywords like "nice to have", "minor", "cosmetic", "low" → Low
- Default → Medium

**Size** — based on scope signals:
- "typo", "copy change", "one line", "config" → XS
- "single file", "small fix", "simple" → S
- "moderate", "several files", "new endpoint" → M
- "complex", "multiple systems", "migration" → L
- "cross-repo", "architecture", "new service" → XL
- Default → M

**Milestone** — match against open milestones:
```bash
gh api repos/$REPO_OWNER/$REPO_NAME/milestones \
  --jq '.[] | select(.state == "open") | {number, title, due_on}'
```
Compare milestone titles against issue title/body keywords. Case-insensitive substring match.

---

## Step 3 — Present Suggestions

Show all suggestions per issue. Wait for user input before applying.

```
🏷️ Issue Triage — N untriaged issues

#601 "Fix slow dashboard loading" — @dev1
  → Type: bug | Priority: High | Size: M | Milestone: feat/performance
  [Apply? Y/n/edit]

#605 "Add dark mode toggle" — unassigned
  → Type: feature | Priority: Low | Size: S | Milestone: —
  [Apply? Y/n/edit]
```

Accepted responses per issue:
- `Y` / `y` / blank → apply all suggestions
- `n` → skip this issue
- `edit` → ask user for correct values, then apply

---

## Step 4 — Apply (write)

On user confirm per issue:

**Apply type label:**
```bash
gh issue edit $ISSUE_NUMBER --repo "$REPO_FULL" --add-label "$TYPE_LABEL"
```

**Set milestone (if matched):**
```bash
gh issue edit $ISSUE_NUMBER --repo "$REPO_FULL" --milestone "$MILESTONE_TITLE"
```

**Set Priority + Size via Projects (if PROJECT_ID detected):**

```bash
# 1. Get issue node_id
ISSUE_NODE_ID=$(gh api repos/$REPO_OWNER/$REPO_NAME/issues/$ISSUE_NUMBER --jq .node_id)

# 2. Add issue to project (idempotent)
ITEM_ID=$(gh api graphql -f query='
  mutation($projectId: ID!, $contentId: ID!) {
    addProjectV2ItemById(input: { projectId: $projectId, contentId: $contentId }) {
      item { id }
    }
  }
' -f projectId="$PROJECT_ID" -f contentId="$ISSUE_NODE_ID" --jq '.data.addProjectV2ItemById.item.id')

# 3. Set Priority
gh api graphql -f query='
  mutation($projectId: ID!, $itemId: ID!, $fieldId: ID!, $value: String!) {
    updateProjectV2ItemFieldValue(input: {
      projectId: $projectId, itemId: $itemId, fieldId: $fieldId,
      value: { singleSelectOptionId: $value }
    }) { projectV2Item { id } }
  }
' -f projectId="$PROJECT_ID" -f itemId="$ITEM_ID" \
  -f fieldId="$PRIORITY_FIELD_ID" -f value="$PRIORITY_OPTION_ID"

# 4. Set Size
gh api graphql -f query='
  mutation($projectId: ID!, $itemId: ID!, $fieldId: ID!, $value: String!) {
    updateProjectV2ItemFieldValue(input: {
      projectId: $projectId, itemId: $itemId, fieldId: $fieldId,
      value: { singleSelectOptionId: $value }
    }) { projectV2Item { id } }
  }
' -f projectId="$PROJECT_ID" -f itemId="$ITEM_ID" \
  -f fieldId="$SIZE_FIELD_ID" -f value="$SIZE_OPTION_ID"
```

After applying, confirm: `✅ #601: bug | High | M | feat/performance — applied`

**Rules:**
- Steps 1–3 are readonly. Step 4 writes only after explicit user confirmation.
- Never auto-apply without asking first
- If Projects board not detected, skip Priority/Size fields — apply label + milestone only
- If no milestones match, leave milestone empty — do not guess
