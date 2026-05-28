---
name: wtb-issue-create
description: "Create a well-researched GitHub issue with codebase analysis, full template, and GitHub Projects field setup. Run from any git repo."
user-invocable: true
---

# /wtb-issue-create

Create a well-researched GitHub issue with codebase analysis and structured template.

## Invocation

```
/wtb-issue-create [description]
```

Optionally attach images to include screenshots in the issue.
Run from within the target git repository directory.

---

## Step 0 — Foundation

Read and execute the shared foundation before proceeding:
1. Read `shared/foundation.md` (relative to this plugin's root directory)
2. Execute sections A–D as described
3. Store: `REPO_FULL`, `REPO_OWNER`, `REPO_NAME`, `PROJECT_ID` (if detected), field IDs

---

## Step 1 — Gather & Clarify

Before writing anything, assess what was provided:

- **If the request is unclear or has multiple valid approaches** → ask up to 4 clarifying questions. Present all options with tradeoffs and a recommendation. Wait for answers before proceeding.
- **If images are attached** → read them immediately with the `Read` tool and incorporate context into the issue.
- **If the request is clear** → proceed to Step 2 directly.

Do NOT create the issue until Step 5.

---

## Step 2 — Research Codebase

Search the **current working directory** (the git repo you're in) to ground the issue in concrete detail.

For the issue topic, identify:
- Exact file paths + line numbers involved
- Current behavior (what the code does now)
- Proposed implementation (what needs to change and where)
- Which files/modules are affected

Use `Grep` and `Read` tools — do NOT guess file locations.

After research, assess **Size**:

| Size | Signals |
|---|---|
| XS | Single file, cosmetic/copy fix, < 10 lines |
| S | 1–2 files, isolated fix, < 50 lines |
| M | 2–5 files, moderate logic change, single area |
| L | 5+ files or multiple subsystems involved |
| XL | Cross-repo, architectural change, new feature with migrations |

Store as `SUGGESTED_SIZE`.

---

## Step 3a — Milestone Match

```bash
gh api repos/$REPO_OWNER/$REPO_NAME/milestones \
  --jq '.[] | select(.state == "open") | {number: .number, title: .title, due_on: .due_on}'
```

Match by comparing milestone titles against the issue title/description keywords:
- Case-insensitive substring match
- If 2+ milestones match → show candidates, ask user to pick
- Skip milestones that are closed or past `due_on`
- If no match → leave milestone empty

---

## Step 3b — Image Upload (if images attached)

If images were provided, upload them using the GitHub REST API:

```bash
TOKEN=$(gh auth token)

# Get or create draft release for issue assets
RELEASE_ID=$(curl -s -H "Authorization: token $TOKEN" \
  "https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/releases" \
  | python3 -c "
import json, sys
releases = json.load(sys.stdin)
draft = [r for r in releases if r.get('draft') and 'issue-assets' in r.get('tag_name','')]
print(draft[0]['id'] if draft else 'NONE')
")

if [ "$RELEASE_ID" = "NONE" ]; then
  RELEASE_ID=$(curl -s -X POST \
    -H "Authorization: token $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"tag_name":"issue-assets","name":"Issue Assets","draft":true,"body":""}' \
    "https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/releases" \
    | python3 -c "import json, sys; print(json.load(sys.stdin)['id'])")
fi

# Upload image — detect content type from extension
EXT="${IMG_PATH##*.}"
case "$EXT" in
  png) CT="image/png" ;;
  gif) CT="image/gif" ;;
  webp) CT="image/webp" ;;
  *) CT="image/jpeg" ;;
esac

IMG_URL=$(curl -s -X POST \
  -H "Authorization: token $TOKEN" \
  -H "Content-Type: $CT" \
  --data-binary @"$IMG_PATH" \
  "https://uploads.github.com/repos/$REPO_OWNER/$REPO_NAME/releases/${RELEASE_ID}/assets?name=$(basename $IMG_PATH)" \
  | python3 -c "import json, sys; print(json.load(sys.stdin).get('browser_download_url', 'UPLOAD_FAILED'))")
```

Store each `IMG_URL` for use in the issue body.

---

## Step 3c — Priority Selection

Prompt the user to select a priority level:

```
Priority? [H]igh / [M]edium (default) / [L]ow
```

Wait for user response:
- `h` / `high` → High
- `m` / `medium` / blank → Medium (default)
- `l` / `low` → Low

Store as `PRIORITY`.

---

## Step 4 — Pre-flight Summary

Present this summary and **wait for explicit confirmation**:

```
📋 Pre-flight Summary

Title:      [Bug] Fix slow dashboard loading
Labels:     bug
Priority:   High (user selected)
Size:       M (auto-suggested, override? reply with size)
Milestone:  feat/performance (#8) — or "none"
Assignee:   — (none)
Images:     2 uploaded — or "none"

Description: [2-3 sentence summary]

[Confirm? Y/n/edit]
```

- `Y` / `y` / blank → proceed to Step 5
- `n` → abort
- `edit` → ask what to change → update → re-display summary

**Issue title prefix** based on type:
- Bug → `[Bug]`
- Feature → `[Feature]`
- Enhancement → `[Enhancement]`
- Tech Debt → `[Tech Debt]`
- Research → `[Research]`

---

## Step 5a — Create Issue

```bash
gh issue create --repo "$REPO_FULL" \
  --title "$ISSUE_TITLE" \
  --body "$ISSUE_BODY" \
  --label "$LABELS" \
  --milestone "$MILESTONE_TITLE"
```

Issue body template:

```markdown
## Summary
[One sentence: what needs to be built or fixed, and why]

**Priority:** [High / Medium / Low]

## Background / Context
[Why this matters — user pain, business value, or tech debt reason]

[Screenshot if applicable: ![description](IMG_URL)]

## Acceptance Criteria
- [ ] [Specific, testable condition 1]
- [ ] [Specific, testable condition 2]
- [ ] [Edge cases handled]

## Out of Scope
- [What this issue explicitly does NOT cover]

## How to Implement
[Concrete steps with file paths and line numbers from Step 2 research]

## Dependencies
- Blocked by: #[issue] / [external system] (or "none")
- Related to: #[issue] (or "none")

## Notes
[Implementation hints, design links, API docs, or open questions]
```

**Label mapping:**
| Title Prefix | Label(s) |
|---|---|
| [Bug] | `bug` |
| [Feature] | `enhancement` |
| [Enhancement] | `enhancement` |
| [Tech Debt] | `tech-debt` |
| [Research] | `research` |

Extract the created issue number from the `gh issue create` output URL.

---

## Step 5b — Set GitHub Projects Fields

Only if `PROJECT_ID` is detected (from Step 0). Skip entirely if no Projects board.

```bash
ISSUE_NUMBER=<from Step 5a>

# 1. Get issue node_id
ISSUE_NODE_ID=$(gh api repos/$REPO_OWNER/$REPO_NAME/issues/$ISSUE_NUMBER --jq .node_id)

# 2. Add issue to project
ITEM_ID=$(gh api graphql -f query='
  mutation($projectId: ID!, $contentId: ID!) {
    addProjectV2ItemById(input: { projectId: $projectId, contentId: $contentId }) {
      item { id }
    }
  }
' -f projectId="$PROJECT_ID" -f contentId="$ISSUE_NODE_ID" --jq '.data.addProjectV2ItemById.item.id')

# 3. Set Priority field
gh api graphql -f query='
  mutation($projectId: ID!, $itemId: ID!, $fieldId: ID!, $value: String!) {
    updateProjectV2ItemFieldValue(input: {
      projectId: $projectId, itemId: $itemId, fieldId: $fieldId,
      value: { singleSelectOptionId: $value }
    }) { projectV2Item { id } }
  }
' -f projectId="$PROJECT_ID" -f itemId="$ITEM_ID" \
  -f fieldId="$PRIORITY_FIELD_ID" -f value="$PRIORITY_OPTION_ID"

# 4. Set Size field
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

After issue is created, display:
```
✅ Issue created: https://github.com/[owner]/[repo]/issues/[number]

Title:    [Bug] Fix slow dashboard loading
Labels:   bug
Priority: High | Size: M — set in GitHub Projects ✅
```

---

## Rules

- **Language:** English only in all issue content
- **No sensitive data:** no tokens, passwords, internal IPs, personal info
- **Solution depth:** always include specific file paths + line numbers from Step 2
- **Ambiguity:** never guess — ask first in Step 1
- **No issue without AC:** untestable issues are non-issues — every issue must have Acceptance Criteria
- **Images:** upload via draft release assets only — never commit to repo
- **Priority is required:** always ask in Step 3c, even if everything else is clear
- **Projects fields are optional:** if no Projects board detected, skip Step 5b gracefully
