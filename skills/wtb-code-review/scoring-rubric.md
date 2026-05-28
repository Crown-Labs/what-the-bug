# WTB Code Review — Scoring Rubric

Referenced by `/wtb-code-review` SKILL.md in Step 4.

---

## Components

Three scores, each 0.0–10.0 (one decimal place):

### Code Quality
- Readability and consistency with surrounding code
- Maintainability — can someone else modify this later?
- Simplicity — is the solution as simple as it can be?
- Clarity of intent — does the code communicate what it does?
- Dead code — no unused imports, variables, or functions
- Naming — descriptive, consistent with codebase conventions
- Duplication — DRY without over-abstracting

### Correctness
- Logic accuracy — does it do what it claims?
- Null/edge-case handling — what happens with empty, nil, zero, max values?
- Exception handling — are errors caught and handled appropriately?
- Async/await correctness — proper handling of promises, no unhandled rejections
- Race conditions — concurrent access to shared state
- Resource disposal — files, connections, subscriptions properly closed
- Resource leaks — memory, file handles, event listeners
- Testing quality — tests verify real behavior, not just mocks; edge cases covered

### Production Readiness
- Security — injection prevention, secret leakage, auth bypass, input validation
- Performance — hot paths optimized, no N+1 queries, no unnecessary allocations, no I/O blocking in hot paths
- Layering/architecture — changes respect existing boundaries and patterns
- Observability — appropriate logging, error reporting, metrics where needed

---

## Score Anchors

| Score | Meaning |
|-------|---------|
| 10.0 | Exemplary — nothing to flag |
| 9.0 | High quality — only Minor polish suggestions |
| 8.0 | Solid — Minor issues present but no risk of bugs or maintainability problems |
| 7.0 | Acceptable — at least one Medium concern in this component |
| 6.0 and below | At least one Critical concern, or multiple Medium concerns |

---

## Cap Rules

These are hard caps — the component score cannot exceed the cap regardless of how good the rest of the code is.

| Condition | Cap |
|-----------|-----|
| Single 🔴 Critical issue in this component | **6.0** |
| Multiple 🟡 Medium issues in this component | **7.0** |
| Single 🟡 Medium issue | No cap (score based on overall quality) |
| Only 🟢 Minor issues | No cap |

---

## Issue Severity Definitions

### 🔴 Critical — must fix before merge
- Security vulnerabilities (injection, secret leakage, auth bypass)
- Data loss risk (unprotected deletes, missing transactions)
- Logic errors that break core functionality
- Race conditions that cause corruption

### 🟡 Medium — should fix
- Risk of bugs in non-critical paths
- Missing error handling that could cause silent failures
- Maintainability problems that will compound (god functions, unclear abstractions)
- Performance issues in frequently-used paths
- Missing input validation at system boundaries

### 🟢 Minor — nice to have
- Naming improvements
- Style consistency polish
- Minor optimization opportunities
- Additional test coverage suggestions
- Documentation improvements

---

## Overall Score

**Overall = mean(Code Quality, Correctness, Production Readiness)**, rounded to 1 decimal.

Example:
- Code Quality: 8.5
- Correctness: 7.0 (one Medium issue: missing null check)
- Production Readiness: 9.0
- **Overall: 8.2**

---

## Examples by Component

### Code Quality examples
- 🔴 Critical: Function duplicates 200 lines of existing utility with subtle differences
- 🟡 Medium: Method is 150+ lines with nested conditionals — hard to follow
- 🟢 Minor: Variable `d` could be renamed to `discountRate`

### Correctness examples
- 🔴 Critical: Async function missing `await` on database write — data silently lost
- 🟡 Medium: No null check on API response — will throw in production if service returns 204
- 🟢 Minor: Edge case not tested: what happens with an empty array?
- 🟡 Medium: New API endpoint has no tests — untested code path in production
- 🟢 Minor: Test exists but only covers happy path — missing edge case for empty input

### Production Readiness examples
- 🔴 Critical: User input concatenated into SQL query — injection vulnerability
- 🟡 Medium: N+1 query in a loop — will be slow at scale
- 🟢 Minor: Could add a log line for easier debugging of this flow
