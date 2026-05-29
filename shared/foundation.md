# WTB Shared Foundation

Every skill reads this file as **Step 0** before executing its own steps. Execute sections A through D in order. Store all extracted variables for use in subsequent steps.

---

## A. Repo Detection

Auto-detect the current repository from git remote. Fall back to asking the user.

```bash
REMOTE_URL=$(git remote get-url origin 2>/dev/null || echo "NO_REMOTE")
```

If `REMOTE_URL` is `NO_REMOTE`:
- Stop and ask: "Which repository? (format: owner/repo)"
- Store the user's answer as `REPO_FULL`

If `REMOTE_URL` is a valid URL, parse it:

```bash
# HTTPS: https://github.com/owner/repo.git → owner/repo
# SSH:   git@github.com:owner/repo.git     → owner/repo
REPO_FULL=$(echo "$REMOTE_URL" \
  | sed 's|.*github\.com[:/]||' \
  | sed 's|\.git$||')
REPO_OWNER=$(echo "$REPO_FULL" | cut -d'/' -f1)
REPO_NAME=$(echo "$REPO_FULL" | cut -d'/' -f2)
```

If parsing fails (empty `REPO_OWNER` or `REPO_NAME`):
- Stop and ask: "Could not detect repo from remote URL `$REMOTE_URL`. Which repository? (format: owner/repo)"

**Stored variables:** `REPO_FULL`, `REPO_OWNER`, `REPO_NAME`

---

## B. gh CLI Auth Check

Verify `gh` is installed and authenticated.

```bash
if ! command -v gh &>/dev/null; then
  echo "ERROR: gh CLI not installed. Install it: https://cli.github.com"
  exit 1
fi

gh auth status 2>&1
```

If `gh auth status` exits non-zero:
- Stop: "Not authenticated. Run `gh auth login` first."

**Stored variable:** `GH_USER` (the authenticated username, from `gh api user --jq .login`)

---

## C. Timezone & Date Helpers

All date calculations use Thailand timezone (GMT+7). Working days are Monday through Friday.

**Constants:**
- `TZ=Asia/Bangkok`
- Working days: Mon–Fri
- Date display format: `May 28, 2026`

**Helper: Calculate "yesterday" (skip weekends):**

```bash
YESTERDAY=$(TZ=Asia/Bangkok python3 -c "
from datetime import datetime, timedelta
today = datetime.now()
if today.weekday() == 0:    # Monday → Friday
    offset = 3
elif today.weekday() == 6:  # Sunday → Friday
    offset = 2
else:
    offset = 1
print((today - timedelta(days=offset)).strftime('%Y-%m-%d'))
")
```

**Helper: Calculate "this week" range (Mon–Fri):**

```bash
read WEEK_START WEEK_END < <(TZ=Asia/Bangkok python3 -c "
from datetime import datetime, timedelta
today = datetime.now()
monday = today - timedelta(days=today.weekday())
friday = monday + timedelta(days=4)
print(monday.strftime('%Y-%m-%d'), friday.strftime('%Y-%m-%d'))
")
```

**Helper: Calculate "next week" range:**

```bash
read NEXT_MON NEXT_FRI < <(TZ=Asia/Bangkok python3 -c "
from datetime import datetime, timedelta
today = datetime.now()
next_monday = today + timedelta(days=(7 - today.weekday()))
next_friday = next_monday + timedelta(days=4)
print(next_monday.strftime('%Y-%m-%d'), next_friday.strftime('%Y-%m-%d'))
")
```

**Helper: Format date for display:**

```bash
# Input: 2026-05-28 → Output: May 28, 2026
FORMAT_DATE=$(TZ=Asia/Bangkok date -d "$DATE_VAR" +"%b %-d, %Y")
```

**Helper: Today's date:**

```bash
TODAY=$(TZ=Asia/Bangkok date +%Y-%m-%d)
```

**Stored variables:** `TODAY`, `YESTERDAY`, `WEEK_START`, `WEEK_END`, `NEXT_MON`, `NEXT_FRI`

---

## D. GitHub Projects Detection (optional)

Attempt to find a ProjectV2 board linked to the repository. This is optional — if no board is found, skills fall back to issue state + labels.

The plugin uses the first ProjectV2 board linked to the repository.

### D1. Query Projects

```bash
PROJECT_DATA=$(gh api graphql -f query='
  query($owner: String!, $repo: String!) {
    repository(owner: $owner, name: $repo) {
      projectsV2(first: 10) {
        nodes {
          id
          title
          fields(first: 20) {
            nodes {
              ... on ProjectV2SingleSelectField {
                id
                name
                options { id name }
              }
            }
          }
        }
      }
    }
  }
' -f owner="$REPO_OWNER" -f repo="$REPO_NAME" 2>/dev/null || echo "NO_PROJECTS")
```

If `PROJECT_DATA` is `NO_PROJECTS` or contains no ProjectV2 nodes:
- Set `PROJECT_ID=""` (empty — no Projects board)
- Skills that depend on Projects fields gracefully skip those features
- Skip D2 entirely

### D2. Parse Fields

Select the preferred board and extract field IDs using **case-insensitive contains** matching. This handles boards where fields are named "Project Priority" instead of "Priority", etc.

```bash
# Select project board — use the first linked board
PROJECT_JSON=$(echo "$PROJECT_DATA" | python3 -c "
import json, sys
data = json.load(sys.stdin)
nodes = data['data']['repository']['projectsV2']['nodes']
if not nodes:
    print('{}')
    sys.exit()
print(json.dumps(nodes[0]))
")

PROJECT_ID=$(echo "$PROJECT_JSON" | python3 -c "import json,sys; print(json.load(sys.stdin).get('id',''))")

# Parse all single-select fields into a lookup
# Match by case-insensitive contains: "priority", "size", "status"
read PRIORITY_FIELD_ID PRIORITY_FIELD_NAME PRIORITY_OPTIONS \
     SIZE_FIELD_ID SIZE_FIELD_NAME SIZE_OPTIONS \
     STATUS_FIELD_ID STATUS_FIELD_NAME STATUS_OPTIONS \
< <(echo "$PROJECT_JSON" | python3 -c "
import json, sys

project = json.load(sys.stdin)
fields = [f for f in project.get('fields',{}).get('nodes',[]) if f.get('name')]

def find_field(keyword):
    for f in fields:
        if keyword in f['name'].lower():
            opts = {o['name']: o['id'] for o in f.get('options',[])}
            return f['id'], f['name'], json.dumps(opts)
    return '', '', '{}'

p_id, p_name, p_opts = find_field('priority')
s_id, s_name, s_opts = find_field('size')
st_id, st_name, st_opts = find_field('status')

# Output as tab-separated for read
print(f'{p_id}\t{p_name}\t{p_opts}\t{s_id}\t{s_name}\t{s_opts}\t{st_id}\t{st_name}\t{st_opts}')
")
```

If `PROJECT_ID` is empty after parsing, treat as no Projects board.

**Stored variables:**
- `PROJECT_ID` — the ProjectV2 node ID
- `PRIORITY_FIELD_ID`, `PRIORITY_FIELD_NAME`, `PRIORITY_OPTIONS` — field ID, actual field name (e.g. "Project Priority"), JSON map `{"High":"abc","Medium":"def","Low":"ghi"}`
- `SIZE_FIELD_ID`, `SIZE_FIELD_NAME`, `SIZE_OPTIONS` — same pattern
- `STATUS_FIELD_ID`, `STATUS_FIELD_NAME`, `STATUS_OPTIONS` — same pattern

All empty strings if no Projects board detected.

### D3. Helper: Look Up Option ID

Use this helper whenever you need to convert a user-facing value (e.g. "High", "XS") to its Projects option ID:

```bash
# Usage: lookup_option "$OPTIONS_JSON" "$VALUE"
# Returns: option ID or empty string
lookup_option() {
  echo "$1" | python3 -c "import json,sys; print(json.load(sys.stdin).get('$2',''))"
}

# Example:
PRIORITY_OPTION_ID=$(lookup_option "$PRIORITY_OPTIONS" "$PRIORITY")
SIZE_OPTION_ID=$(lookup_option "$SIZE_OPTIONS" "$SUGGESTED_SIZE")
```

---

## Shared Conventions

### Output Rules

1. Always show assignee when available: `— @username`
2. Always show Priority + Size when available: `[High/S]`
3. Omit empty sections — do not show "Blocked: (none)"
4. Date format: `May 28, 2026`
5. No markdown tables in output — use bullet lists (mobile-friendly)
6. Emoji section headers for scannability

### Error Handling

1. `gh` not installed → "Install gh CLI: https://cli.github.com"
2. `gh` not authenticated → "Run `gh auth login` first"
3. Not a git repo → "Navigate to a git repository first"
4. No remote → "No git remote configured"
5. API rate limit → "GitHub API rate limit hit. Try again in X minutes."
6. Projects board not found → graceful fallback (not a hard error)

### Labels Convention

Type labels (pick one per issue):
- `bug` — defect or regression
- `feature` — new functionality
- `enhancement` — improvement to existing feature
- `tech-debt` — refactor/cleanup
- `research` — spike or investigation
- `blocked` — waiting on dependency
