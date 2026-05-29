# What The Bug (WTB) 🐛

A Claude Code plugin for GitHub issue tracking, PR management, and code review. Built for teams using `gh` CLI.

## Installation

```bash
# Add the marketplace
/plugin marketplace add crown-labs/what-the-bug

# Install the plugin
/plugin install what-the-bug
```

**Requirements:**
- [GitHub CLI (`gh`)](https://cli.github.com) installed and authenticated (`gh auth login`)
- Git repository with a GitHub remote

## Skills

### Reporting (readonly)

| Skill | Purpose |
|-------|---------|
| `/wtb-daily-standup` | Morning standup — closed yesterday, in progress, blocked, candidates |
| `/wtb-weekly-report` | End-of-week summary — throughput, completed, carry-over, blockers |
| `/wtb-weekly-plan` | Plan next week — carry-over + prioritized backlog recommendations |
| `/wtb-blocked-alert` | Scan for blocked issues, stale items, stale PRs, merge conflicts |
| `/wtb-pr-status` | Open PRs snapshot grouped by review status with CI results |

### Triage (read + write on confirm)

| Skill | Purpose |
|-------|---------|
| `/wtb-issue-triage` | Auto-classify untriaged issues — suggest type, priority, size, milestone |

### Actions (write)

| Skill | Purpose |
|-------|---------|
| `/wtb-issue-create` | Create a well-researched GitHub issue with codebase analysis |
| `/wtb-pr-create` | Create a PR with mandatory issue linking and auto-generated description |
| `/wtb-code-review` | Code review with scoring rubric — local, self-review, or peer review |

## Features

- **Multi-repo support** — auto-detects repo from `git remote`, works in any cloned repo
- **GitHub Projects v2** — reads Priority, Size, Status fields when a Projects board is detected
- **No extra dependencies** — uses `gh` CLI + `git` only
- **Mobile-friendly output** — bullet lists, emoji headers, no markdown tables
- **Scoring rubric** — structured code review with 3 components scored 0.0–10.0

## Example Usage

### Create Issues

```
/wtb-issue-create [description]
```

**From a bug report:**
```
/wtb-issue-create Users report 500 error on /api/checkout when cart has more than 50 items
```

**From a feature request:**
```
/wtb-issue-create Add dark mode toggle to the settings page with system preference detection
```

### Create Pull Requests

```
/wtb-pr-create [optional title or description] [--draft]
```

**Auto-detect title from commits/branch:**
```
/wtb-pr-create
```

**With a custom title:**
```
/wtb-pr-create Add rate limiting to public API endpoints
```

**Create a draft PR:**
```
/wtb-pr-create --draft
```

**Draft with custom title:**
```
/wtb-pr-create Add rate limiting to public API endpoints --draft
```

### Code Review

```
/wtb-code-review [--local] [--pr N] [--peer]
```

**Auto-detect mode (local changes or current branch PR):**
```
/wtb-code-review
```

**Review local uncommitted/staged changes (pre-PR):**
```
/wtb-code-review --local
```

**Review a specific PR (self-review if you're the author):**
```
/wtb-code-review --pr 42
```

**Peer review a teammate's PR:**
```
/wtb-code-review --pr 42 --peer
```

## Report Skills

### Daily Standup

```
/wtb-daily-standup
```

Example output:
```
📅 Daily Standup — Thu May 29

✅ Done Yesterday
- #412 Fix session timeout on mobile — @alice (merged)
- #415 Update API rate limit docs — @bob (merged)

🔄 In Progress Today
- #420 Add billing page — @alice
- #423 Refactor auth middleware — @bob

🚧 Blocked
- #418 Payment gateway integration — @alice — waiting on 3rd party API key

📋 Candidates (unassigned, ready to pick up)
- #430 [High/S] Add CSRF protection to forms
- #431 [Medium/XS] Update footer copyright year
```

### Weekly Report

```
/wtb-weekly-report
```

Example output:
```
📊 Weekly Report — May 26–May 30

📈 Throughput
- Closed: 8 issues, 6 PRs merged
- Opened: 5 new issues
- Net: +3 resolved

✅ Completed
- #412 Fix session timeout on mobile [High/S] — @alice
- #415 Update API rate limit docs [Low/XS] — @bob
- #416 Add input validation to signup [Medium/M] — @alice

🔄 Carry-over (still in progress)
- #420 Add billing page [High/L] — @alice
- #423 Refactor auth middleware [Medium/M] — @bob

🚧 Blockers This Week
- #418 waited 4 days on 3rd party API key — @alice

👥 Contributor Activity
- @alice: 5 issues closed, 4 PRs merged
- @bob: 3 issues closed, 2 PRs merged
```

### Weekly Plan

```
/wtb-weekly-plan
```

Example output:
```
📋 Weekly Plan — Jun 2–Jun 6

🔄 Carry-over (from this week)
1. #420 Add billing page [High/L] — @alice continues
2. #423 Refactor auth middleware [Medium/M] — @bob continues

📥 Recommended from Backlog
3. #430 Add CSRF protection [High/S] — unassigned
4. #431 Update footer copyright [Medium/XS] — unassigned
5. #435 Add search to dashboard [Low/M] — unassigned

⚠️ Dependencies
- #435 depends on #423 (auth refactor)

📅 Milestone Deadlines
- #430 in milestone "v2.1-security" due Jun 15

💡 Notes
- Total planned: 2 carry-over + 3 new = 5 issues
- Estimated capacity: 2 devs × 5 days = ~20 story points
```

### Blocked Alert

```
/wtb-blocked-alert
```

Example output:
```
🚨 Blocked & Stale Alert

🔴 Blocked Issues
- #418 Payment gateway integration — @alice — blocked 4 days (since May 26)

🟡 Stale Issues (no activity > 3 working days)
- #410 Add export to CSV — @bob — last update May 22

⏳ PRs Awaiting Review > 2 working days
- PR #89 by @bob — review requested from @alice

💥 PRs with Merge Conflicts
- PR #87 by @alice — conflicts with main
```

### Issue Triage

```
/wtb-issue-triage
```

Example output:
```
🏷️ Issue Triage — 3 untriaged issues

#440 "Dashboard loads slowly with 1000+ records"
  → Type: bug | Priority: High | Size: M | Milestone: v2.1-performance
  [Apply? Y/n/edit]

#441 "Add dark mode toggle to settings"
  → Type: feature | Priority: Low | Size: S | Milestone: —
  [Apply? Y/n/edit]

#442 "Refactor database connection pooling"
  → Type: tech-debt | Priority: Medium | Size: L | Milestone: v2.2
  [Apply? Y/n/edit]
```

## How It Works

All skills share a common foundation (`shared/foundation.md`) that handles:
1. **Repo detection** — parses `git remote` URL (HTTPS or SSH)
2. **Auth check** — verifies `gh` is authenticated
3. **Timezone** — all dates in GMT+7 (Thailand), Mon–Fri working days
4. **GitHub Projects** — optional detection of ProjectV2 board with graceful fallback

## GitHub Projects Integration

If your repo has a GitHub Projects board, WTB skills automatically detect and use:
- **Priority** field (High / Medium / Low)
- **Size** field (XS / S / M / L / XL)
- **Status** field (In Progress / Done / etc.)

Uses the first ProjectV2 board linked to the repository. If no board is found, skills fall back to issue labels and state.

## Code Review Scoring

`/wtb-code-review` uses a 3-component rubric:

- **Code Quality** (readability, maintainability, naming, duplication)
- **Correctness** (logic, null handling, async, resource leaks)
- **Production Readiness** (security, performance, architecture)

Each scored 0.0–10.0. Cap rules: Critical issue → max 6.0, multiple Mediums → max 7.0.

See `skills/wtb-code-review/scoring-rubric.md` for the full rubric.

## License

MIT
