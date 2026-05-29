---
name: wtb-pr-create
description: "Create a pull request with mandatory issue linking, diff analysis, auto-generated description, and pre-flight summary. Run from any git repo."
user-invocable: true
---

# /wtb-pr-create

Create a pull request for the current working directory following a 5-step workflow.
All operations use `gh` CLI and `git`.

## Invocation

```
/wtb-pr-create [optional description or override title]
```

Run from within the target git repository directory (e.g. `projects/getpod`).

---

## Step 0 — Foundation

Read and execute the shared foundation before proceeding:
1. Read `shared/foundation.md` (relative to this plugin's root directory)
2. Execute sections A–B as described (repo detection + gh auth check)
3. Store: `REPO_FULL`, `REPO_OWNER`, `REPO_NAME`

Note: Steps C (timezone) and D (Projects) are not needed for PR creation.

---

## Step 1 — Pre-flight Git Check

Run from the repository root. Detect the working directory state and handle all cases
autonomously before proceeding.

### 1a. Get current branch

```bash
CURRENT_BRANCH=$(git branch --show-current)
echo "Current branch: $CURRENT_BRANCH"
```

If `CURRENT_BRANCH` is empty (detached HEAD state) → stop: "Detached HEAD state — checkout a named branch before creating a PR."

### 1b. Check working tree status

```bash
git status --short
```

Interpret the output:

| Status | Action |
|---|---|
| Nothing (empty output) | Proceed — check for commits below |
| Staged files (first column non-space, e.g. `A `, `M `) | Ask: "You have staged files. Commit them before creating the PR? [Y/n]" |
| Unstaged changes only (second column non-space, e.g. ` M`, ` D`) | Warn: "Unstaged changes detected — these won't be in the PR." then continue |
| Both staged + unstaged | Handle staged first (ask to commit), then warn about unstaged |

**If user says yes to commit staged files:**
Ask for a commit message → run:
```bash
git commit -m "$USER_MSG"
```

If `git commit` exits non-zero → stop and display the error: "Commit failed. Fix the issue and re-run /wtb-pr-create."

**If user says no:** Continue with current commits.

### 1c. Check if there are commits to PR

```bash
# Determine target branch for comparison (default: develop, or main if develop doesn't exist)
TARGET=$(git branch -r | grep -q 'origin/develop' && echo "develop" || echo "main")
BASE_REMOTE="origin/$TARGET"

# Count commits ahead
COMMITS_AHEAD=$(git log ${BASE_REMOTE}..HEAD --oneline 2>/dev/null | wc -l | tr -d ' ')
echo "Commits ahead: $COMMITS_AHEAD"
```

If `COMMITS_AHEAD` is 0 → stop: "No new commits compared to `$TARGET`. Nothing to PR."

### 1d. Check if branch is pushed to remote

```bash
REMOTE_EXISTS=$(git ls-remote --heads origin "$CURRENT_BRANCH" 2>/dev/null | wc -l | tr -d ' ')
if [ "$REMOTE_EXISTS" = "0" ]; then
  NEEDS_PUSH=true
  UNPUSHED=0
else
  UNPUSHED=$(git log origin/${CURRENT_BRANCH}..HEAD --oneline | wc -l | tr -d ' ')
  if [ "$UNPUSHED" -gt "0" ]; then
    NEEDS_PUSH=true
  else
    NEEDS_PUSH=false
  fi
fi
```
Do not push yet — defer to Step 5.

Store all values for use in later steps:
- `CURRENT_BRANCH`
- `TARGET` (default base branch)
- `COMMITS_AHEAD`
- `NEEDS_PUSH`

---

## Step 2 — Analyze Diff + Generate PR Description

### 2a. Get commit log since base

```bash
git log ${BASE_REMOTE}..HEAD --oneline
```

Example output:
```
a1b2c3d feat: add keyboard shortcut for sidebar toggle
e4f5g6h fix: prevent stale VM status after reconnect
```

### 2b. Get changed files

```bash
git diff ${BASE_REMOTE}..HEAD --stat --no-color
```

### 2c. Detect related issue references

**Layer 1 — Scan commit messages** for patterns `closes #NNN`, `fixes #NNN`, `resolves #NNN` (case-insensitive):

```bash
git log ${BASE_REMOTE}..HEAD --format="%s %b" | grep -oiE '(closes|fixes|resolves) #[0-9]+' | sort -u
```

**Layer 2 — Semantic search** (run only if Layer 1 returns nothing):
- Extract keywords from branch name and commit subjects (e.g. `sidebar-keyboard-shortcut` → keywords: `sidebar`, `keyboard`, `shortcut`)
- Search open GitHub issues:

```bash
REPO="$REPO_FULL"

# KEYWORDS: strip type prefix from branch name (feat/, fix/, etc.), split on - and _,
# drop common stop words (add, fix, update, feat, the, a, an, for, of)
# Example: feat/sidebar-keyboard-shortcut → sidebar keyboard shortcut
KEYWORDS=$(echo "$CURRENT_BRANCH" \
  | sed 's|^[a-z]*/||' \
  | tr '-_' ' ' \
  | tr '[:upper:]' '[:lower:]' \
  | tr ' ' '\n' \
  | grep -vE '^(add|fix|update|feat|feature|the|a|an|for|of|and|to|with|in|on)$' \
  | tr '\n' ' ' \
  | sed 's/ *$//')

MATCHED=$(gh issue list --repo "$REPO_FULL" --state open --limit 50 --json number,title \
  | KEYWORDS="$KEYWORDS" python3 -c "
import json, sys, os
issues = json.load(sys.stdin)
keywords = os.environ.get('KEYWORDS', '').lower().split()
if not keywords:
    sys.exit(0)
matches = [i for i in issues if any(k in i['title'].lower() for k in keywords)]
for m in matches[:5]:
    print(f\"#{m['number']}: {m['title']}\")
")
```

**Layer 3 — Confidence classification:**
- Layer 1 found issues → `CONFIDENCE=high`, store as `AUTO_ISSUES` (parse issue numbers: `echo "$LAYER1_RESULTS" | grep -oE '#[0-9]+' | tr -d '#'`)
- `MATCHED` has 1–2 lines → `CONFIDENCE=medium`, store as `CANDIDATE_ISSUES`
- `MATCHED` is empty or has 3+ lines → `CONFIDENCE=low`

### 2d. Mandatory Issue Confirmation

**This step is always required — a PR without a linked issue cannot be created.**

Based on confidence level:

**CONFIDENCE=high (commit references found):**
Present auto-detected issues and ask to confirm or add more:
```
Found related issues from commits: Closes #612, Closes #598
Add more? (enter issue numbers separated by commas, or press Enter to confirm)
```
Wait for user response:
- Blank / Enter → use `AUTO_ISSUES` as-is
- Numbers provided → append to `AUTO_ISSUES`

Store final list as `LINKED_ISSUES` (same as `AUTO_ISSUES` after any user additions).

**CONFIDENCE=medium (semantic matches found):**
Present candidates and ask user to confirm:
```
Possible related issues:
  #610: Responsive Design Audit — mobile UX
  #598: Onboarding: add pod setup step before provisioning
Which issues does this PR address? (enter numbers, e.g. 610,598 — required)
```
Wait for user input. **Do not proceed if blank.**

**CONFIDENCE=low (nothing found):**
Ask directly:
```
No related issues found automatically.
Which GitHub issue(s) does this PR address? (required — enter numbers, e.g. 610,598)
```
Wait for user input. **Do not proceed if blank.**

Store confirmed list as `LINKED_ISSUES` — an array of issue numbers, e.g. `[612, 598]`.

**Block if `LINKED_ISSUES` is empty — do not proceed to Step 3.**

Format for PR body: `Closes #612, Closes #598` (one `Closes #NNN` per issue, comma-separated on one line).

### 2e. Auto-assign linked issues

Get the current authenticated GitHub user:

```bash
GH_USER=$(gh api user --jq '.login')
echo "Current user: $GH_USER"
```

For each issue in `LINKED_ISSUES`, check if it has an assignee. If not, assign the current user:

```bash
for ISSUE_NUM in $LINKED_ISSUES; do
  ASSIGNEE=$(gh issue view "$ISSUE_NUM" --repo "$REPO_FULL" --json assignees --jq '.assignees[0].login // empty')
  if [ -z "$ASSIGNEE" ]; then
    gh issue edit "$ISSUE_NUM" --repo "$REPO_FULL" --add-assignee "$GH_USER" && \
      echo "Assigned @$GH_USER to #$ISSUE_NUM" || \
      echo "Warning: Failed to assign #$ISSUE_NUM (continuing)"
  else
    echo "#$ISSUE_NUM already assigned to @$ASSIGNEE"
  fi
done
```

This step is best-effort — if assignment fails (e.g. permissions), warn but continue to Step 2f.

### 2f. Derive PR title

Priority order:
1. If user provided a description/title override at invocation → use that
2. If single commit → use commit subject: strip the conventional commit prefix (everything up to and including the `:` and space, e.g. `feat: `, `fix: `), then capitalise the first letter of each word. Example: `feat: add keyboard shortcut` → `Add Keyboard Shortcut`
3. If multiple commits → derive from branch name: strip the type prefix (e.g. `feat/`, `fix/`, `refactor/`), replace `-` and `_` with spaces, then capitalise the first letter of each word. Example: `feat/sidebar-keyboard-shortcut` → `Sidebar Keyboard Shortcut`

Apply type prefix based on branch name or commit type:
- Branch starts with `feat/` or `feature/` OR commit starts with `feat:` → `[Feature]`
- Branch starts with `fix/` or `bugfix/` OR commit starts with `fix:` → `[Fix]`
- Branch starts with `refactor/` → `[Refactor]`
- Branch starts with `docs/` → `[Docs]`
- Branch starts with `chore/` → `[Chore]`
- Default → `[Feature]`

Example: `feat/sidebar-keyboard-shortcut` → `[Feature] Sidebar Keyboard Shortcut`

Store as `PR_TITLE`.

### 2g. Determine change type (for PR body checkbox)

Based on type prefix detected above:
- `[Fix]` → check "Bug fix" box
- `[Feature]` → check "New feature" box
- `[Refactor]` → check "Refactor / Tech debt" box
- `[Docs]` → check "Documentation" box
- `[Chore]` → check "Enhancement" box (no dedicated checkbox — closest match)
- Default → check "Enhancement" box

### 2h. Build PR body

Using this template, fill in all sections:

```markdown
## Summary
[1–2 sentences: what changed and why]

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Enhancement
- [ ] Refactor / Tech debt
- [ ] Documentation

## Related Issues
Closes #[issue1], Closes #[issue2]

## Changes Made
- `[file/module]`: [what changed]

## How to Test
1. [Step 1]
2. [Step 2]
Expected: [result]

## Checklist
- [ ] Reviewed my own code
- [ ] Tests pass (if applicable)
- [ ] No console errors
```

Fill in:
- **Summary:** 1–2 sentences derived from commit messages or branch name
- **Type of Change:** pre-check the matching checkbox from 2f (replace `[ ]` with `[x]` for the matching type)
- **Related Issues:** formatted `LINKED_ISSUES` from Step 2d — e.g. `Closes #612, Closes #598`
- **Changes Made:** list the first 10 files from `git diff --stat` output. If there are more than 10 changed files, add a final line `...and N more files` where N is the remaining count.
- **How to Test:** generic placeholder — "1. Checkout this branch. 2. [describe key scenario based on PR title]. Expected: [changes are visible/functional]."
- **Checklist:** all unchecked by default

Store as `PR_BODY`.

---

## Step 3 — Branch & Target Detection

### 3a. Detect base branch

Use this priority order:
1. Check git branch config: `git config branch.${CURRENT_BRANCH}.gh-merge-base 2>/dev/null`
   - If non-empty → use that value as `BASE_BRANCH`
2. Check if `origin/develop` exists: `git ls-remote --heads origin develop | wc -l | tr -d ' '`
   - If result is `1` → `BASE_BRANCH=develop`
3. Fall back to `BASE_BRANCH=main`

```bash
BASE_BRANCH=$(git config "branch.${CURRENT_BRANCH}.gh-merge-base" 2>/dev/null)

if [ -z "$BASE_BRANCH" ]; then
  DEVELOP_EXISTS=$(git ls-remote --heads origin develop 2>/dev/null | wc -l | tr -d ' ')
  if [ "$DEVELOP_EXISTS" -gt 0 ] 2>/dev/null; then
    BASE_BRANCH=develop
  else
    BASE_BRANCH=main
  fi
fi

echo "Base branch: $BASE_BRANCH"
```

Update `BASE_REMOTE` to match: `BASE_REMOTE="origin/$BASE_BRANCH"`
(This overrides the `TARGET`/`BASE_REMOTE` set in Step 1c, which used a faster heuristic.)

### 3b. Handle same-branch situation

```bash
if [ "$CURRENT_BRANCH" = "$BASE_BRANCH" ]; then
  echo "SAME_BRANCH"
fi
```

If `CURRENT_BRANCH == BASE_BRANCH`:
- Inform user: "You're currently on '`$BASE_BRANCH`' — can't PR a branch into itself."
- Auto-generate a branch name from `PR_TITLE` using this snippet:

```bash
# Determine type prefix from PR_TITLE
case "$PR_TITLE" in
  \[Fix\]*) BRANCH_PREFIX="fix/" ;;
  \[Refactor\]*) BRANCH_PREFIX="refactor/" ;;
  \[Docs\]*) BRANCH_PREFIX="docs/" ;;
  \[Chore\]*) BRANCH_PREFIX="chore/" ;;
  *) BRANCH_PREFIX="feat/" ;;
esac

# Strip title prefix, lowercase, replace spaces with dashes, strip special chars
STRIPPED=$(echo "$PR_TITLE" | sed 's/^\[[^]]*\] *//')
BRANCH_BODY=$(echo "$STRIPPED" \
  | tr '[:upper:]' '[:lower:]' \
  | tr ' ' '-' \
  | sed 's/[^a-z0-9-]//g' \
  | sed 's/--*/-/g' \
  | sed 's/^-//;s/-$//')

AUTO_BRANCH="${BRANCH_PREFIX}${BRANCH_BODY}"
```

  - Example: `PR_TITLE = "[Feature] Sidebar Keyboard Shortcut"` → `feat/sidebar-keyboard-shortcut`
- Present to user:
  ```
  Auto-generated branch: feat/sidebar-keyboard-shortcut
  Use this? [Y/n/enter custom name]
  ```
Assign the chosen name to `NEW_BRANCH` before running git commands:
- If `y` / `Y` / blank → `NEW_BRANCH=$AUTO_BRANCH`
- If custom text entered → `NEW_BRANCH=<user input>`

Run:
```bash
git checkout -b "$NEW_BRANCH" || {
  echo "ERROR: Branch '$NEW_BRANCH' already exists or checkout failed."
  echo "Rename the branch or delete the local branch and retry."
  exit 1
}

git push -u origin "$NEW_BRANCH" || {
  echo "ERROR: Push to origin failed."
  echo "Check your permissions and network connection, then retry."
  exit 1
}
```

- Update `CURRENT_BRANCH` to the new branch name
- Set `NEEDS_PUSH=false` (already pushed above)

### 3c. Fetch reviewer suggestions (best-effort)

Check for CODEOWNERS file:
```bash
CODEOWNERS_CONTENT=$(cat .github/CODEOWNERS 2>/dev/null || cat CODEOWNERS 2>/dev/null || echo "NONE")
```

If `CODEOWNERS_CONTENT` is not `"NONE"`:
- For each file in the `git diff --stat` output, look up the matching CODEOWNERS rule.
  Use last-matching-rule semantics (the last line in CODEOWNERS whose pattern is a substring
  of the file path wins). If glob matching is complex, fall back to: take the first
  `@username` found on any CODEOWNERS line whose pattern appears in the file path.
- Extract the GitHub usernames (tokens starting with `@` after the path pattern).
- Collect up to 2 unique usernames, store as `SUGGESTED_REVIEWERS`.

If `CODEOWNERS_CONTENT` is `"NONE"` or no matching rules found:
- `SUGGESTED_REVIEWERS=none`

---

## Step 4 — Pre-flight Summary

Present this summary and **wait for explicit confirmation** before proceeding to Step 5.

When displaying the summary, replace all `$VARIABLE` tokens with their actual values — do not show the literal variable names.

```
─────────────────────────────────────────
PR Pre-flight Summary
─────────────────────────────────────────
Title:        $PR_TITLE
Base branch:  $BASE_BRANCH
Head branch:  $CURRENT_BRANCH
Commits:      $COMMITS_AHEAD commit(s)
Push needed:  $NEEDS_PUSH

Issues:       $LINKED_ISSUES (formatted as: Closes #612, Closes #598)
Reviewers:    $SUGGESTED_REVIEWERS (or "none detected")

─────────────────────────────────────────
PR Description Preview:
[output of: echo "$PR_BODY" | head -10]
─────────────────────────────────────────
Ready to create PR? [yes / no / edit title]
```

Accepted responses:
- `yes` / `y` / blank Enter → proceed to Step 5
- `no` / `n` → abort: "PR creation cancelled."
- `edit title` → ask: "Enter new title:" → wait for input → update `PR_TITLE` → re-display the full summary above (loop)
- Any other string that starts with a capital letter and contains spaces (and is NOT a variant of `yes`/`no`/`edit title`) → treat as new title override → update `PR_TITLE` → re-display summary (loop)

**Do NOT proceed to Step 5 until the user confirms with `yes` or equivalent.**

---

## Step 5 — Create Pull Request

### 5a. Push branch if needed

If `NEEDS_PUSH=true`:

```bash
git push -u origin "$CURRENT_BRANCH" || {
  echo "ERROR: Push failed."
  echo "Error details above. Resolve the issue and re-run /wtb-pr-create."
  exit 1
}
```

If `NEEDS_PUSH=false`, skip this step.

### 5b. Create the PR

Write `PR_BODY` to a temp file to safely handle multi-line content:

```bash
PR_BODY_FILE=$(mktemp)
printf '%s' "$PR_BODY" > "$PR_BODY_FILE"
```

Build reviewer flags (space-separated list of `@username` entries in `SUGGESTED_REVIEWERS`):

```bash
REVIEWER_FLAGS=""
if [ "$SUGGESTED_REVIEWERS" != "none" ]; then
  for REVIEWER in $SUGGESTED_REVIEWERS; do
    REVIEWER_FLAGS="$REVIEWER_FLAGS --reviewer $REVIEWER"
  done
fi
```

Build draft flag:

```bash
# Set DRAFT_FLAG=true if the user's invocation string contained the literal token --draft
# (e.g. the user typed /wtb-pr-create --draft or /wtb-pr-create "some title" --draft)
# Otherwise DRAFT_FLAG=false
DRAFT_FLAG_ARG=""
if [ "$DRAFT_FLAG" = "true" ]; then
  DRAFT_FLAG_ARG="--draft"
fi
```

Execute PR creation, capturing output and stderr separately:

```bash
ERR_FILE=$(mktemp)
PR_OUTPUT=$(gh pr create \
  --repo "$REPO_FULL" \
  --title "$PR_TITLE" \
  --body-file "$PR_BODY_FILE" \
  --base "$BASE_BRANCH" \
  --head "$CURRENT_BRANCH" \
  $REVIEWER_FLAGS \
  $DRAFT_FLAG_ARG \
  2>"$ERR_FILE")
PR_STATUS=$?
rm -f "$PR_BODY_FILE"
```

### 5c. Handle result

**On success** (`PR_STATUS=0`):

Extract the PR URL (last line of output — `gh pr create` prints the URL as its final line):

```bash
PR_URL=$(echo "$PR_OUTPUT" | tail -1)
```

Display to user:

```
✅ PR created: $PR_URL

Title:   $PR_TITLE
Base:    $BASE_BRANCH ← $CURRENT_BRANCH
Issues:  [formatted LINKED_ISSUES: Closes #612, Closes #598]
```

Then run:

```bash
rm -f "$ERR_FILE"
```

**On failure** (`PR_STATUS != 0`):

```
❌ PR creation failed.
Error:
[output of: cat "$ERR_FILE"]

Resolve the issue above and re-run /wtb-pr-create.
```

Do not retry automatically. Clean up:

```bash
rm -f "$ERR_FILE"
```

---

## Rules

- **Tools:** `gh` CLI for all GitHub operations, `git` for local repo operations
- **Working directory:** all `git` and `gh` commands must run from inside the target repo directory (CWD at invocation time)
- **No push without push-check:** always check `NEEDS_PUSH` in Step 1 before touching the remote
- **Branch safety:** never force-push; never delete remote branches
- **Draft mode:** if the invocation string contains the literal token `--draft` (e.g. `/wtb-pr-create --draft` or `/wtb-pr-create "My Title" --draft`), set `DRAFT_FLAG=true` and pass `--draft` to `gh pr create`. Detect this at the start of execution before Step 1.
- **Multi-repo:** this skill operates on one repo at a time — the CWD at invocation time
- **Language:** English only in all PR titles and body content
- **Confirmation required:** never skip Step 4 pre-flight, even if all values are auto-detected
- **Issue tagging is mandatory:** every PR must have at least one `Closes #NNN` in the body. Step 2d must always run and must always collect at least one issue number. If `LINKED_ISSUES` is empty after Step 2d, block and re-ask — do not proceed to Step 3.
- **Error handling:** when any `git` or `gh` command fails, surface the full error output to the user and stop — do not continue with partial state
