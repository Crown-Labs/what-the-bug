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

### Issue Management

```
/wtb-issue-create [description]
/wtb-issue-triage
```

**Create an issue from a bug report:**
```
/wtb-issue-create Users report 500 error on /api/checkout when cart has more than 50 items
```

**Create an issue from a feature request:**
```
/wtb-issue-create Add dark mode toggle to the settings page with system preference detection
```

**Triage unclassified issues:**
```
/wtb-issue-triage
```

### Pull Requests

```
/wtb-pr-create [optional title or description] [--draft]
/wtb-pr-status
```

**Create a PR (auto-detect title from commits/branch):**
```
/wtb-pr-create
```

**Create a PR with a custom title:**
```
/wtb-pr-create Add rate limiting to public API endpoints
```

**Create a draft PR:**
```
/wtb-pr-create --draft
```

**Create a draft PR with a custom title:**
```
/wtb-pr-create Add rate limiting to public API endpoints --draft
```

**Check status of all open PRs:**
```
/wtb-pr-status
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

### Reports

```
/wtb-daily-standup
/wtb-weekly-report
/wtb-weekly-plan
/wtb-blocked-alert
```

**Morning standup summary:**
```
/wtb-daily-standup
```

**End-of-week report:**
```
/wtb-weekly-report
```

**Plan next week's work:**
```
/wtb-weekly-plan
```

**Scan for blocked issues and stale PRs:**
```
/wtb-blocked-alert
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
