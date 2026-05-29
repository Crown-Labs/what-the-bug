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

**Create an issue from a bug description:**
```
/wtb-issue-create Users report 500 error on /api/checkout when cart has more than 50 items
```

**Create a PR after fixing a bug:**
```
/wtb-pr-create Fixes the cart overflow bug by adding pagination to the checkout query
```

**Review your own changes before pushing:**
```
/wtb-code-review
```

**Run morning standup:**
```
/wtb-daily-standup
```

**Triage unclassified issues:**
```
/wtb-issue-triage
```

**Check open PR status:**
```
/wtb-pr-status
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
