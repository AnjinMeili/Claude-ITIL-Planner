# Commit Protocol Reference

Full guide to the Conventional Commits standard as applied in the planitil workflow.

---

## The Format

```
<type>(<scope>): <subject>

[body]

[footer(s)]
```

All four parts have rules. Type and subject are required. Scope, body, and footers are optional but have defined meanings when present.

---

## Types

| Type | When to use |
|---|---|
| `feat` | New user-visible behavior — something the user can now do that they could not before |
| `fix` | Correction of a bug — incorrect behavior is now correct |
| `docs` | Documentation only — no code behavior changes |
| `refactor` | Code restructuring that preserves behavior — no new features, no bug fixes |
| `test` | Adding, modifying, or fixing tests — no production code changes |
| `chore` | Build system, CI, dependency updates, project tooling |
| `perf` | Measurable performance improvement — include the measurement in the body |
| `style` | Formatting, whitespace, semicolons — no logic changes (rare; usually handled by formatter) |
| `revert` | Reverting a previous commit — reference the reverted commit SHA in the footer |

Do not invent new types. If the change does not fit a type, it is either two commits or it belongs in `chore`.

---

## Scope

Scope identifies the module, component, or area of the codebase the change affects. It appears in parentheses after the type:

```
feat(auth): add JWT refresh token support
fix(database): handle null on user lookup
refactor(cli): extract argument parsing into its own module
```

Scope is optional but strongly recommended on any project with more than one logical component. Omitting scope on a large project makes the log unreadable.

Use consistent scope names. Define them in DEV-SPEC.md and do not deviate. `api` vs `API` vs `server` for the same component creates log noise.

---

## Subject Line Rules

- Imperative mood: "add", "fix", "extract", "remove" — not "added", "fixes", "extracts"
- 72 characters maximum including type and scope prefix
- No period at the end
- Lowercase after the colon: `feat(auth): add ...` not `feat(auth): Add ...`
- Describe the *what*, not the *how* — the diff shows how

---

## Body Rules

The body explains *why* the change was made. It is optional but expected when the reason for the change is not obvious from the subject.

- Separate from subject with a blank line
- Wrap at 72 characters
- Address the business or technical reason, not the implementation detail
- Reference issue numbers here if not in the footer

**When to include a body:**

- The change reverses a previous decision — explain why the decision changed
- The change is non-obvious — explain what would go wrong without it
- The change has a workaround quality — reference the DEBT.md entry

---

## Breaking Change Notation

A breaking change is any change that requires callers to update their code. Mark it with `!` after the type/scope and add a `BREAKING CHANGE:` footer:

```
feat(api)!: require Authorization header on all endpoints

Previously, unauthenticated requests returned data with reduced scope.
This was a security gap identified in the Q2 audit.

BREAKING CHANGE: All requests now require a valid Bearer token.
Unauthenticated requests return HTTP 401.
```

The `BREAKING CHANGE:` footer triggers a major version bump in semantic versioning tooling.

---

## Squash vs Merge Strategy

**Squash on merge** (default recommendation): All commits on a feature branch are squashed into a single commit on `main`. The result is a clean, readable `main` history where each commit represents a complete feature or fix. The branch history is preserved in the PR record.

Use squash when:
- Branch commits are "WIP", "fix typo", "address review feedback" — not meaningful individually
- The feature is small enough that one commit is accurate

**Merge commit** (preserve history): All branch commits appear individually on `main`. Use this when:
- Each branch commit is independently meaningful (e.g., a four-step database migration)
- The team needs the intermediate states in `git bisect`
- The branch is long-lived and collaborative

**Rebase** (linear history): Branch commits are replayed on top of `main` without a merge commit. Produces clean linear history. Requires that all branch authors are comfortable with rebase workflow. Do not rebase shared branches.

Document which strategy is used for which context in DEV-SPEC.md. Never mix strategies on the same repository without a rule for when to use each.

---

## 10 Annotated Examples

### Example 1 — Good: clear feat with scope

```
feat(auth): add OAuth2 login via Google
```

Clear type, scoped to auth, imperative subject, describes what was added.

### Example 2 — Good: fix with body explaining why

```
fix(sessions): expire sessions after 24h of inactivity

Sessions were persisting indefinitely. Under load, the session store
was exhausting memory. The 24h expiry matches the UX contract stated
in the onboarding copy.
```

### Example 3 — Good: breaking change correctly marked

```
feat(config)!: rename DATABASE_URL to DB_CONNECTION_STRING

BREAKING CHANGE: The DATABASE_URL environment variable is no longer
read. Update .env and all deployment configurations to use
DB_CONNECTION_STRING instead.
```

### Example 4 — Good: chore with version pin

```
chore(deps): pin requests to 2.31.0

requests 2.32.0 introduced a behavior change in redirect handling
that breaks our OAuth callback flow. Pinned until upstream fix is
available (see DEBT-012).
```

### Example 5 — Good: refactor with scope, no behavior change

```
refactor(reports): extract date range filtering into ReportQuery

No behavior changes. Extraction enables unit testing of the filter
logic in isolation.
```

### Example 6 — Bad: past tense

```
feat(auth): added JWT support
```

Fix: `feat(auth): add JWT support`

### Example 7 — Bad: vague subject

```
fix: fixed the bug
```

Fix: `fix(payments): handle declined card response from Stripe`

### Example 8 — Bad: multiple changes in one commit

```
feat(api): add user endpoint, fix login redirect, update README
```

Fix: Split into three commits. Each type prefix signals the problem — you cannot have `feat`, `fix`, and `docs` in one subject line.

### Example 9 — Bad: body describes what, not why

```
refactor(models): moved User class to its own file

Moved the User class from models.py to user.py. Updated all imports.
```

Fix: The body should explain *why*: "models.py had grown to 600 lines. Extracted User to enable independent testing and to reduce merge conflicts on a heavily-edited file."

### Example 10 — Bad: breaking change not marked

```
feat(api): require authentication on all endpoints
```

Fix: `feat(api)!: require authentication on all endpoints` with a `BREAKING CHANGE:` footer. Not marking a breaking change is a contract violation for any downstream consumer.
