---
name: wtb-code-review
description: "Review code changes with scoring rubric. Supports local (pre-PR), self-review, and peer review modes. Posts results to GitHub PR. Run from any git repo."
user-invocable: true
---

# /wtb-code-review

Review code changes with a structured scoring rubric. Supports local changes (pre-PR), self-review, and peer review.

## Invocation

```
/wtb-code-review                    → auto-detect mode
/wtb-code-review --local            → force local changes mode
/wtb-code-review --pr 123           → review specific PR (auto-detect self vs peer)
/wtb-code-review --pr 123 --peer    → force peer review mode
```

Run from within any git repository with a GitHub remote.

---

## Step 0 — Foundation

Read and execute the shared foundation before proceeding:
1. Read `shared/foundation.md` (relative to this plugin's root directory)
2. Execute sections A–B as described (repo detection + gh auth)
3. Store: `REPO_FULL`, `GH_USER`

---

## Step 1 — Mode Detection

Parse invocation flags to determine the review mode:

| Flag | Mode |
|------|------|
| `--local` | Local changes (pre-PR) — review uncommitted/staged changes |
| `--pr N` | PR review — auto-detect self vs peer from author |
| `--pr N --peer` | Force peer review mode |
| No flags | Auto-detect (see below) |

**Auto-detect logic (no flags):**

1. Check for uncommitted changes:
```bash
LOCAL_CHANGES=$(git diff --stat 2>/dev/null; git diff --cached --stat 2>/dev/null)
```

2. Check for open PR from current branch:
```bash
CURRENT_BRANCH=$(git branch --show-current)
PR_DATA=$(gh pr view "$CURRENT_BRANCH" --repo "$REPO_FULL" --json number,author 2>/dev/null || echo "NO_PR")
```

3. Decision:
   - If `LOCAL_CHANGES` is non-empty → **Local mode**
   - Else if `PR_DATA` contains a PR number → extract `PR_NUMBER` and `PR_AUTHOR`, determine self/peer
   - Else → ask user: "No local changes or open PR found. Provide a PR number or run from a branch with changes."

**Self vs Peer detection (for PR mode):**
```bash
PR_AUTHOR=$(echo "$PR_DATA" | jq -r '.author.login')
if [ "$PR_AUTHOR" = "$GH_USER" ]; then
  MODE="self-review"
else
  MODE="peer-review"
fi
```

Store: `MODE` (local / self-review / peer-review), `PR_NUMBER` (if applicable)

---

## Step 2A — Local Changes Review (pre-PR mode)

Gather all local diffs:

```bash
# Unstaged changes
UNSTAGED_DIFF=$(git diff)

# Staged changes
STAGED_DIFF=$(git diff --cached)

# Combine
FULL_DIFF="${STAGED_DIFF}${UNSTAGED_DIFF}"
```

If `FULL_DIFF` is empty → "No local changes to review."

Use the combined diff for analysis in Step 3.

---

## Step 2B — PR Review (self-review or peer-review mode)

Gather PR data:

```bash
# Full diff
gh pr diff $PR_NUMBER --repo "$REPO_FULL"

# PR metadata
gh pr view $PR_NUMBER --repo "$REPO_FULL" \
  --json title,body,author,labels,milestone,headRefName

# Linked issues (parse from PR body for "Closes #N", "Fixes #N", etc.)
PR_BODY=$(gh pr view $PR_NUMBER --repo "$REPO_FULL" --json body --jq .body)
LINKED_ISSUES=$(echo "$PR_BODY" | grep -oiE '(closes|fixes|resolves) #[0-9]+' | grep -oE '[0-9]+' | sort -u)
```

If `LINKED_ISSUES` is not empty, fetch their Acceptance Criteria:
```bash
for ISSUE_NUM in $LINKED_ISSUES; do
  gh issue view $ISSUE_NUM --repo "$REPO_FULL" --json body --jq .body
done
```

Extract AC sections from issue bodies for use in Step 3 (AC Coverage check).

---

## Step 3 — Analysis

Review the diff across 6 dimensions:

### 3a. Logic & Correctness
- Bugs, edge cases, null handling
- Race conditions, async/await issues
- Off-by-one errors, boundary conditions
- Exception handling gaps

### 3b. Security
- Injection vulnerabilities (SQL, XSS, command)
- Secret leakage in code or logs
- Auth/authz bypass risks
- Input validation at system boundaries

### 3c. Performance
- Hot path efficiency
- N+1 queries, unnecessary allocations
- I/O blocking in async contexts
- Caching opportunities missed

### 3d. Maintainability
- Naming clarity and consistency
- Code duplication
- Complexity (deeply nested logic, god functions)
- Consistency with surrounding codebase patterns

### 3e. AC Coverage (if linked issues exist)
- For each Acceptance Criterion in linked issues, check if the diff covers it
- Mark each AC as covered (✅) or missing (⚠️)

### 3f. Testing
- Do tests verify real behavior, not just mock responses?
- Edge cases covered in tests? (empty, null, boundary, error states)
- Integration tests for critical paths? (API calls, DB operations)
- Test naming — clear what's being tested and expected outcome?
- No tests deleted or skipped without justification
- New code paths have corresponding test coverage

**Note:** Testing findings feed into the **Correctness** scoring component.

**For each issue found, record:**
- Severity: 🔴 Critical / 🟡 Medium / 🟢 Minor
- File path + line number
- Description of the issue
- Suggested fix

---

## Step 4 — Scoring

Read `scoring-rubric.md` (in this skill's directory) for the full rubric.

Calculate three component scores (0.0–10.0, one decimal):

| Component | Covers |
|-----------|--------|
| **Code Quality** | Readability, maintainability, simplicity, naming, duplication |
| **Correctness** | Logic accuracy, null/edge handling, async correctness, resource leaks |
| **Production Readiness** | Security, performance, architecture adherence, observability |

**Apply cap rules:**
- A single 🔴 Critical issue caps the relevant component at **6.0 or below**
- Multiple 🟡 Medium issues in the same component cap it at **7.0 or below**

**Overall score** = mean of 3 components, rounded to 1 decimal.

**Score anchors:**
| Score | Meaning |
|-------|---------|
| 10.0 | Exemplary — nothing to flag |
| 9.0 | High quality — only Minor suggestions |
| 8.0 | Solid — Minor issues, no bug risk |
| 7.0 | Acceptable — at least one Medium concern |
| 6.0- | Critical concern or multiple Mediums |

---

## Step 5 — Output

Format the review results:

```
🔍 Code Review — [#PR123 feat: billing page (self-review) | local changes]

📊 Score: 8.3 / 10.0
- Code Quality:        8.5
- Correctness:         8.0
- Production Readiness: 8.5

💪 Strengths
- Clean separation of billing logic from UI — easy to test independently
- Good use of TypeScript generics for payment types
- Error boundary properly catches render failures

🔴 Critical (0)
(none)

🟡 Medium (2)
1. src/billing.ts:45 — No error handling on payment API call
   → Wrap in try/catch, handle timeout + retry
2. src/billing.ts:102 — User input not sanitized before DB query
   → Use parameterized query

🟢 Minor (3)
1. src/billing.ts:12 — Variable name `d` unclear → `discountRate`
2. src/utils.ts:88 — Duplicate helper, same as `formatCurrency` in shared/
3. src/billing.test.ts — Missing edge case: zero amount

✅ AC Coverage (from linked issues)
- [x] User can view billing history (#456 AC 1)
- [x] User can download invoice PDF (#456 AC 2)
- [ ] ⚠️ Error state not tested (#456 AC 3)

💡 Suggestions
- Consider adding integration test for payment flow
- billing.ts is 280 lines — consider splitting

🏁 Verdict: ✅ Ready to merge
```

**Strengths section rules:**
- Lists 2-4 specific things done well in the code
- Must be concrete and specific (file/pattern references), not generic praise
- Always appears before issues — acknowledge good work first
- If nothing noteworthy → show `- Solid implementation overall` (minimum 1 line)

**Verdict logic** (derive from score + critical count):
- `✅ Ready to merge` — Score >= 8.0 AND no Critical issues
- `⚠️ Ready to merge with fixes` — Score 7.0-7.9 AND no Critical issues (list the Medium items to address)
- `🚫 Not ready — requires changes` — Score < 7.0 OR has Critical issues (list blockers)

**Conditional sections:**
- If no Critical issues → show `🔴 Critical (0)` with "(none)"
- If no linked issues → omit "AC Coverage" section entirely
- "Suggestions" section is optional — only include if there are broad recommendations beyond specific issues

---

## Step 6 — Post Review to GitHub

**Applies to:** Self-review mode and Peer review mode (any mode with `PR_NUMBER`).
**Does NOT apply to:** Local changes mode (no PR exists).

The review body content is the same as Step 5 output.

### Peer Review Mode

**Gate logic:**
- Score >= 8.0 AND no 🔴 Critical issues → offer all options:
  ```
  Post review to GitHub? [approve / comment / request-changes / skip]
  ```
- Score < 8.0 OR has 🔴 Critical issues → no approve option:
  - Has Critical → "Recommend: request changes"
  - No Critical but < 8.0 → "Recommend: comment with feedback"
  ```
  Post review to GitHub? [comment / request-changes / skip]
  ```

**Execute on confirm:**
```bash
# Approve
gh pr review $PR_NUMBER --repo "$REPO_FULL" --approve --body "$REVIEW_BODY"

# Comment only
gh pr review $PR_NUMBER --repo "$REPO_FULL" --comment --body "$REVIEW_BODY"

# Request changes
gh pr review $PR_NUMBER --repo "$REPO_FULL" --request-changes --body "$REVIEW_BODY"
```

### Self-Review Mode

**Gate logic:**
```
Post review as comment to GitHub PR? [yes / skip]
```

Self-review always posts as a regular comment (cannot approve or request-changes on your own PR):

```bash
gh pr comment $PR_NUMBER --repo "$REPO_FULL" --body "$REVIEW_BODY"
```

---

## Rules

- **Scoring rubric:** always read `scoring-rubric.md` before scoring — do not invent criteria
- **File paths:** always include exact file path + line number for each issue found
- **Suggested fixes:** always include a concrete suggestion, not just "fix this"
- **Cap rules are mandatory:** Critical → 6.0 cap, multiple Mediums → 7.0 cap
- **No approve with Critical issues:** never offer approve when 🔴 issues exist
- **Local mode is output-only:** never attempt to post to GitHub in local mode
- **AC Coverage requires linked issues:** if no issues linked to the PR, omit the AC section
- **Strengths section is mandatory:** always acknowledge 2-4 specific positive aspects before listing issues
- **Verdict is mandatory:** always end output with a clear Ready/Not Ready verdict derived from score + critical count

---

## Appendix: Receiving Review Feedback

Guidance for when YOU receive review feedback on your code. This is not part of the review workflow — it's a reference for handling incoming feedback with technical rigor.

### The Response Pattern

```
1. READ — Complete all feedback without reacting
2. UNDERSTAND — Restate each item in your own words (or ask)
3. VERIFY — Check against codebase reality before implementing
4. EVALUATE — Is this technically sound for THIS codebase?
5. IMPLEMENT — One item at a time, test each
```

### Verify Before Implementing

Before acting on any feedback:
- Check if the suggestion breaks existing functionality
- Check if there's a reason for the current implementation (git blame, comments)
- Check if the suggestion works across all use cases, not just the reviewer's scenario
- If you can't verify → say so: "I can't verify this without [X]. Should I investigate?"

### When to Push Back

Push back when:
- Suggestion breaks existing tests or functionality
- Reviewer lacks full context of the change
- Violates YAGNI — adding unused features "for completeness"
- Technically incorrect for this stack/version
- Conflicts with architectural decisions already made

How to push back:
- Use technical reasoning, not defensiveness
- Reference working tests or code
- Ask specific questions: "Would this break X?" not "I don't think that's right"

### Implementation Order

When addressing multiple feedback items:
1. Clarify anything unclear FIRST — don't implement partial understanding
2. Blocking issues (security, breaks, data loss)
3. Simple fixes (naming, imports, formatting)
4. Complex fixes (refactoring, logic changes)
5. Test each fix individually
6. Verify no regressions after all fixes

### YAGNI Check

If a reviewer suggests "implementing properly" or "adding for completeness":
- Search the codebase for actual usage
- If unused → question whether it's needed: "Nothing calls this. Remove it (YAGNI)?"
- If used → then implement properly

### Acknowledging Correct Feedback

When feedback IS correct:
- State the fix: "Fixed. [Brief description of what changed]"
- Or just fix it — the code shows you heard the feedback
- No performative agreement ("Great point!", "You're absolutely right!")
- Actions over words
