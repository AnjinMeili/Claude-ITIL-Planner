---
name: dev-discipline
description: |
  Produces DEV-SPEC.md — the code standards, commit protocol, review criteria, and debt management policy for a project. Establishes the "how we work" contract before implementation begins.

  <example>
  User says "what are our coding standards" or "set up the dev workflow" — invoke dev-discipline to produce DEV-SPEC.md covering all four sections.
  </example>

  <example>
  Before the first implementation task, after ARCH.md is confirmed — invoke dev-discipline to lock in the working agreement so all implementation agents share the same standards.
  </example>
version: 1.0.0
---

# dev-discipline

## What This Skill Produces

Produce `DEV-SPEC.md` at `.planning/DEV-SPEC.md`. The file contains four sections: Code Standards, Commit Protocol, Review Criteria, and Debt Management Policy. Together these four sections form the working agreement that every contributor — human and agent — follows for the life of the project.

Note: `code-reviewer` checks both `.planning/DEV-SPEC.md` and the project root for backward compatibility. The canonical location is `.planning/DEV-SPEC.md`.

Do not invent standards on an existing project. Use dev-discipline to document what is already practiced, not to impose a new regime. On a greenfield project, define standards that the team can realistically follow with the tools and workflow already in place.

---

## Section 1 — Code Standards

### Language and Framework Conventions

Follow the canonical style guide for the project's primary language without modification:

- **Python:** PEP 8 enforced via `ruff` or `flake8`. Type annotations required on all public functions and methods. Docstrings in Google style.
- **TypeScript/JavaScript:** ESLint with the project's agreed rule set. Prettier for formatting. No `any` types in TypeScript without an explicit `// eslint-disable` comment explaining why.
- **Go:** `gofmt` enforced. No unused imports or variables (the compiler enforces this; never suppress).
- **Other languages:** Document the equivalent canonical formatter and linter in DEV-SPEC.md. If no canonical tool exists, pick one and pin the version.

### Naming Rules

Apply these rules universally regardless of language:

- Name variables, functions, and types for what they represent, not how they are implemented. `user_session` not `us` or `obj`.
- Boolean variables and functions take a predicate form: `is_active`, `has_permission`, `can_write`.
- Functions that cause side effects use verb phrases: `save_record`, `send_notification`, `delete_user`.
- Functions that return values without side effects use noun phrases or question forms: `current_timestamp()`, `active_sessions()`.
- Constants are UPPER_SNAKE_CASE in all languages.
- Do not abbreviate unless the abbreviation is universally understood in the domain (e.g., `id`, `url`, `http`).

### File Organization Pattern

Organize files by feature, not by type. A `users/` directory contains models, services, routes, and tests for user functionality — not a flat `models/` directory containing all models for the entire system. This rule applies at the module level; within a module, follow the language convention.

Maximum file length: 400 lines. If a file exceeds this, extract a new module. This is a forcing function for decomposition, not a hard technical limit.

### Line Length

120 characters maximum. Configure the formatter to enforce this; do not rely on manual discipline.

### Import Ordering

Follow the language canonical order (stdlib → third-party → local). Configure the linter to enforce order automatically. Never mix ordered and unordered import blocks.

### Type Annotation Requirements

All public interfaces carry type annotations. Private helpers may omit them when the type is obvious from context. Return types are always annotated on public functions — never rely on inference at the call boundary.

### Security Rules

Apply these rules without exception:

- **No hardcoded secrets.** No API keys, passwords, tokens, connection strings, or private certificates in source code. Use environment variables loaded via `python-dotenv`, `dotenv` (Node), or the equivalent for the runtime.
- **dotenv pattern:** Maintain a `.env.example` file at the project root listing every required variable with a placeholder value and a comment describing its purpose. Never commit `.env`.
- **No `eval` or dynamic code execution** without explicit justification in a comment.
- **Input validation at system boundaries.** Validate and sanitize all external input at the entry point — do not pass raw external data into internal functions.

---

## Section 2 — Commit Protocol

### Format

Follow Conventional Commits. Every commit message takes the form:

```
<type>(<scope>): <subject>

[optional body]

[optional footer]
```

**Types:**

| Type | Use when |
|---|---|
| `feat` | Introducing new user-visible functionality |
| `fix` | Correcting a bug that caused incorrect behavior |
| `docs` | Changes to documentation only |
| `refactor` | Code change that neither adds a feature nor fixes a bug |
| `test` | Adding or modifying tests only |
| `chore` | Build system, dependency updates, CI configuration |
| `perf` | Performance improvement with measurable evidence |

**Subject line rules:**

- Imperative mood: "add auth middleware" not "added auth middleware" or "adds auth middleware"
- 72 characters maximum
- No period at the end
- Lowercase after the colon

**Body rules (when used):**

- Separate from subject with a blank line
- Explain *why* the change was made, not *what* was changed (the diff shows what)
- Wrap at 72 characters

**Breaking changes:**

Append `!` after the type/scope and add a `BREAKING CHANGE:` footer explaining what callers must update:

```
feat(api)!: require authentication on all endpoints

BREAKING CHANGE: Unauthenticated requests now return 401.
Clients must include an Authorization header.
```

### One Logical Change Per Commit

Each commit must represent one coherent, reversible unit of work. A commit that "adds the user model, fixes the login bug, and updates the README" is three commits. If reverting this commit would undo more than one thing, split it.

### No --no-verify Bypasses

Never use `--no-verify` to skip pre-commit hooks. If a hook is failing, fix the underlying issue. A hook that is consistently bypassed is not a hook — remove it from the configuration and document why.

### Squash vs Merge Strategy

Squash feature branch commits on merge to keep `main` history readable. Preserve individual commits only when the branch history itself has documentary value (e.g., a multi-step refactor where each step is independently meaningful).

---

## Section 3 — Review Criteria

### What Gets Reviewed

Mandatory review applies to:

- All code touching shared state (database schema changes, shared cache, global configuration)
- All new external dependencies (any new entry in `requirements.txt`, `package.json`, `go.mod`, etc.)
- All security-adjacent code (authentication, authorization, input validation, encryption, secrets handling)
- Any change that modifies a public API contract

Self-review of clearly isolated changes (a single function rename, a comment correction) is acceptable when the change carries no semantic risk.

### Tiered Feedback Format

Record every review comment with its tier:

**Critical** — blocks merge. Must be resolved before the PR can land.
- Security vulnerability or secret exposure
- Data loss risk (missing transaction, incorrect delete, wrong index)
- Broken contract (interface change that breaks downstream callers)
- Failing test in the PR's own test suite

**Important** — strongly recommended. Reviewer expects a response (fix or explanation).
- Missing error handling on external calls (database, HTTP, file I/O)
- Missing test coverage for the new code path
- Logic error that does not currently cause data loss but could under load or edge conditions
- Dependency added without a clear ownership/maintenance plan

**Suggestion** — advisory. No response required; author's judgment prevails.
- Naming clarity improvements
- Readability restructuring that does not change behavior
- Performance optimization with no urgency
- Alternative implementation approach worth considering

**Positive Note** — required. Every review ends with at least one genuine positive observation. Feedback without acknowledgment of what works breeds defensiveness.

### What Blocks Merge

Only Critical items block merge. Important items require a response but not necessarily a fix — the author may explain why the current approach is acceptable. Suggestion items are informational.

A review with zero Critical items is a green review, regardless of the number of Important or Suggestion items.

---

## Section 4 — Debt Management Policy

### What Qualifies as Debt

Debt is a deferred decision or incomplete implementation that is consciously carried forward. Three categories:

**Intentional debt** — a deliberate trade-off. "We used an in-process job queue for MVP; we will migrate to a proper queue when job volume exceeds 1,000/day." The trade-off is understood and the trigger for resolution is defined.

**Unintentional debt** — a gap discovered after the fact. "This function works for the current data shape but will break if a user has more than one active session." No explicit trade-off was made; the gap was revealed by usage or review.

**Bit rot** — previously correct code that is now incorrect due to dependency changes, infrastructure evolution, or shifting requirements. "This API client was written against v2 of the vendor API; they have deprecated v2."

### Recording Debt: DEBT.md

Maintain a `DEBT.md` file at the project root. Each entry follows this format:

```markdown
## DEBT-NNN: <short title>

**What:** <one sentence describing the gap or workaround>
**Why deferred:** <one sentence explaining the reason it was not addressed immediately>
**Impact:** <what breaks or degrades if this is never resolved>
**Category:** intentional | unintentional | bit-rot
**Target resolution:** <milestone or date, or "when X condition is met">
**Opened:** YYYY-MM-DD
**Resolved:** — (or date when closed)
```

### Debt Triage Cadence

Review DEBT.md at the start of each milestone. For each open item, answer:

1. Has the trigger condition been met? If yes, schedule resolution in the current milestone.
2. Has the impact increased since the item was opened? If yes, re-evaluate priority.
3. Is the item still relevant? If not, close it with a reason.

### Debt That Is Never Acceptable

The following are not debt — they are defects that block shipping:

- Known security vulnerabilities (CVEs in dependencies, exposed credentials, broken auth)
- Known data corruption risks (missing transactions, incorrect uniqueness constraints, race conditions that corrupt records)

These must be fixed before the code reaches production, regardless of timeline pressure. Recording them in DEBT.md does not make them acceptable to ship.

---

## When NOT to Invoke

Do not run dev-discipline on an existing project to replace established norms. If a team already has a commit convention, a review process, and a debt practice, use dev-discipline to *document* what they do — not to override it. Imposing new standards on an active project mid-stream creates confusion and resentment.

---

## Common Mistakes

**Standards so strict no one follows them.** Line limits of 79 characters in 2025, mandatory docstrings on every private method, zero-tolerance type coverage before any tests pass — these create friction that incentivizes bypass. Set standards at the level the team will actually hold.

**Commit protocol with no enforcement mechanism.** Documenting that all commits must follow Conventional Commits means nothing without a `commitlint` hook or CI check that fails non-conforming commits. Standards without enforcement are aspirations.

**Review criteria that are all advisory.** If nothing blocks merge, reviews are optional reading. At minimum, define the Critical tier and apply it consistently. One clear blocker tier is better than three undefined tiers.

**Debt policy with no resolution timeline.** A DEBT.md entry that says "resolve eventually" will never be resolved. Every entry must carry either a target date or a trigger condition ("when monthly active users exceed 10k").

---

## Worked Example: dev-discipline for the Time-Tracking CLI

The following is a condensed DEV-SPEC.md for the time-tracking CLI project (Python, single developer, no external API integrations at MVP).

```markdown
# DEV-SPEC.md — time-tracker

## Code Standards

Language: Python 3.12+. Formatter: black (line length 100). Linter: ruff.
Type annotations required on all public functions. Docstrings in Google style on
public functions only. Private helpers may omit docstrings.

File organization: flat src/ layout at MVP; split by feature if >5 modules.

Secrets: NEVER hardcode. Use dotenv. Maintain .env.example.

## Commit Protocol

Format: Conventional Commits. Types in use: feat, fix, refactor, test, chore.
Subject line ≤72 chars, imperative mood. Squash on merge to main.
pre-commit hook: commitlint + ruff + black --check.

## Review Criteria

Self-review permitted on solo project. Before merging any branch:
- Critical: no hardcoded secrets, no data-corruption risk in record writes
- Important: all new code paths have at least one test

## Debt Management

DEBT.md at project root. Target resolution dates required on all intentional debt.
Triage at the start of each sprint. Security and data-corruption defects block shipping.
```

---

## Writing the Artifact

When all four sections are complete, write the artifact to disk:

- **Path:** `.planning/DEV-SPEC.md` at the project root
- Confirm with: `DEV-SPEC.md written to .planning/DEV-SPEC.md`

## References

- [`references/commit-protocol.md`](references/commit-protocol.md) — Full Conventional Commits guide with 10 annotated examples (good and bad), body format rules, breaking change notation, squash vs merge strategy guidance.
- [`references/review-criteria.md`](references/review-criteria.md) — Detailed tier definitions with phrasing guidance for each tier, examples of Critical/Important/Suggestion/Positive feedback, and the operational definition of "blocks merge."
- [`references/debt-management.md`](references/debt-management.md) — Debt classification framework, full DEBT.md entry format, the impact × effort triage matrix, and examples of acceptable vs never-acceptable debt.
