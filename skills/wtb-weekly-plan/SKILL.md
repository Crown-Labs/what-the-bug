---
name: wtb-weekly-plan
description: "Plan next week's work from backlog + carry-over. Recommends issues by priority and size. Run on Fridays from any git repo."
user-invocable: true
---

# /wtb-weekly-plan

Plan next week's work — carry-over from this week plus recommended backlog items. Best run on Fridays.

## Invocation

```
/wtb-weekly-plan
```

Run from within any git repository with a GitHub remote.

---

## Step 0 — Foundation

Read and execute the shared foundation before proceeding:
1. Read `shared/foundation.md` (relative to this plugin's root directory)
2. Execute sections A–D as described
3. Store: `REPO_FULL`, `PROJECT_ID` (if detected), `NEXT_MON`, `NEXT_FRI`

---

## Step 1 — Carry-over

Items still in progress from the current week. These get priority in next week's plan.

**If Projects board detected:**
Query items where Status = "In Progress" and issue state is open.

**Fallback:**
Open issues with assignees (proxy for "someone is working on this").

```bash
gh issue list --repo "$REPO_FULL" --state open \
  --json number,title,assignees,labels \
  --jq '.[] | select(.assignees | length > 0)'
```

These items are listed first in the plan — they represent already-committed work.

---

## Step 2 — Backlog Analysis

Open, unassigned issues sorted by Priority and Size.

```bash
gh issue list --repo "$REPO_FULL" --state open \
  --json number,title,assignees,labels,milestone
```

Filter:
- Unassigned (empty assignees)
- Not labeled `blocked`

**If Projects board detected:**
Query Priority and Size fields. Sort: Priority desc (High → Medium → Low → unset), then Size asc (XS → S → M → L → XL) within same priority.

**If no Projects board:**
Sort by creation date (oldest first as proxy for "been waiting longest").

---

## Step 3 — Recommendations

Build the recommended plan:

1. **Carry-over items first** (from Step 1) — already started, should continue
2. **Backlog items** (from Step 2) — recommended based on Priority + Size fit
3. **Flag dependencies:** if any recommended issue body mentions another recommended issue number, note the dependency
4. **Flag milestone deadlines:** if any recommended issue has a milestone with a `due_on` date within the next 2 weeks, highlight it

---

## Step 4 — Present Draft Plan

Show numbered list for user review. The user can adjust (add, remove, reorder) before finalizing.

```
📋 Weekly Plan — [NEXT_MON]–[NEXT_FRI]

🔄 Carry-over (from this week)
1. #789 Refactor auth [Medium/M] — @dev1 continues
2. #101 Payment integration [High/L] — @dev2 continues

📥 Recommended from Backlog
3. #303 Add rate limiting [High/S] — unassigned
4. #404 Improve error messages [Medium/M] — unassigned
5. #505 Update onboarding flow [Medium/S] — unassigned

⚠️ Dependencies
- #404 depends on #789 (auth refactor must complete first)

📅 Milestone Deadlines
- #303 is in milestone "feat/security" due Jun 15

💡 Notes
- Total planned: 2 carry-over + 3 new = 5 issues
- Consider backup task if #101 stays blocked
```

**Rules:**
- This skill is readonly — it does NOT assign issues or move them on the board
- Show Priority + Size: `[High/S]` for all items
- Show assignee: `— @username` or `— unassigned`
- Omit empty sections
- Maximum 10 recommended items to keep the plan focused
