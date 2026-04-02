# Review Criteria Reference

Detailed tier definitions, phrasing guidance, and the operational definition of "blocks merge."

---

## The Four Tiers

### Critical — Blocks Merge

A Critical finding must be resolved before the PR lands. No exceptions. The reviewer does not approve the PR until every Critical item is either fixed or demonstrated to be a false alarm with an explanation.

**What belongs in Critical:**

**Security issues**
- Hardcoded secrets (API keys, tokens, passwords, private keys) anywhere in the diff
- Authentication bypass or privilege escalation
- SQL injection, command injection, path traversal
- Sensitive data logged or exposed in error messages
- Use of a dependency with a known critical CVE

**Data loss risk**
- Missing database transaction on multi-step write operations
- Incorrect delete scope (deletes more than intended)
- Missing uniqueness constraint that could produce duplicate records
- Race condition that could corrupt a record under concurrent writes
- Migration that drops a column or table without a verified backup step

**Broken contract**
- Change to a public function signature without updating all callers
- Change to an API response shape that breaks documented behavior
- Removal of a required configuration key without a migration path
- Interface change that is not marked as a breaking change in the commit message

**Failing tests**
- Any test in the PR's own test suite that was passing before and is now failing
- New code with no tests when the project standard requires test coverage

**How to phrase Critical feedback:**

Lead with the tier, state the specific location, describe the risk:

```
**Critical** — `src/auth/middleware.py:47`
The JWT secret is read from `os.getenv("SECRET")` with no fallback, but
there is no validation that the variable is set. If SECRET is unset,
`jwt.decode` will accept any token. Add a startup check that raises if
SECRET is missing or empty.
```

Do not soften Critical findings. "You might want to consider..." is not appropriate for a merge blocker.

---

### Important — Response Required

An Important finding does not block merge on its own, but the reviewer expects a response: either a fix, or a clear explanation of why the current approach is acceptable. Silence on an Important item is not acceptable.

**What belongs in Important:**

**Missing error handling on external calls**
- Database call with no error path
- HTTP request with no timeout and no exception handling
- File I/O with no handling for missing file, permission error, or disk full

**Test coverage gaps**
- New branching logic (if/else, try/catch) with no test for the failure branch
- New public function with no unit test
- Bug fix with no regression test

**Correctness concerns**
- Logic that works for the current inputs but will fail at boundary conditions
- Off-by-one in a pagination or slicing operation
- Type coercion that silently drops data

**Dependency concerns**
- New dependency added without a comment explaining the choice
- New dependency that duplicates existing functionality already in the project
- New dependency with no clear ownership or maintenance activity in the last 12 months

**How to phrase Important feedback:**

State the tier, location, the concern, and the expected response:

```
**Important** — `src/jobs/processor.py:112`
The call to `db.execute()` has no try/except. If the database is
temporarily unavailable, this will propagate an unhandled exception
to the job runner and halt the queue. Add error handling that logs
the failure and re-queues the job with backoff.

If there is a reason this case is already handled upstream, please
point to where.
```

---

### Suggestion — Advisory

A Suggestion is a genuine improvement that does not need to happen in this PR. The author acknowledges it (no response required) and may act on it at their discretion.

**What belongs in Suggestion:**

**Naming clarity**
- Variable name that could be more descriptive
- Function name that does not reflect current behavior after a refactor
- Abbreviation that is not universally understood in context

**Readability restructuring**
- Long function that could be extracted without changing behavior
- Nested conditionals that could be flattened
- Repeated pattern that could be extracted into a helper

**Performance**
- Optimization opportunity with no current urgency
- Repeated computation that could be memoized
- N+1 query that is acceptable at current scale but worth noting

**Alternative approaches**
- A library that would simplify the implementation
- A pattern the author may not be aware of that would reduce complexity

**How to phrase Suggestion feedback:**

Use a light touch. Explain the benefit without implying the current approach is wrong:

```
**Suggestion** — `src/reports/builder.py:78-94`
This loop builds the output list item by item. A list comprehension
would be more idiomatic Python and slightly faster. Not urgent — just
something to consider when you are next in this file.
```

---

### Positive Note — Required

Every review ends with at least one genuine positive observation. This is not a courtesy convention — it is an accuracy requirement. A review that only records problems is an incomplete picture of the code.

**What counts as a Positive Note:**

- A well-chosen abstraction that will make future changes easier
- A test that is genuinely thorough or creative
- Error handling that is better than the minimum required
- Clear naming that required thought to get right
- Documentation that explains a non-obvious decision

**What does not count:**

- "Looks good overall" (too vague to be useful)
- A positive note on every single line (dilutes meaning)
- Praise for meeting the minimum standard ("the tests pass")

**Example:**

```
**Positive** — The `RetryPolicy` abstraction is a strong choice here.
Extracting the retry logic makes it independently testable and means
the processor does not need to know about backoff behavior. This will
pay off when we need to tune retry parameters per job type.
```

---

## Operational Definition of "Blocks Merge"

"Blocks merge" means: the reviewer will not approve the PR until the item is resolved. In practice:

1. Reviewer leaves a Critical comment and requests changes (does not approve).
2. Author fixes the Critical item and responds on the comment thread.
3. Reviewer re-reviews the fix. If resolved, approves. If not resolved, re-requests changes.

Important and Suggestion items do not trigger a re-request. The reviewer approves after Critical items are cleared even if Important items have open responses.

A PR with zero Critical items is a green review. A green review does not mean the PR is perfect — it means it is safe to merge.

---

## Review Checklist

Use this as a fast pass before marking a PR reviewed:

- [ ] Checked for hardcoded secrets (grep for key patterns if not automated)
- [ ] Checked all multi-step writes for transactions
- [ ] Checked all public API changes for contract compliance
- [ ] Checked all external calls for error handling
- [ ] Checked new code paths for test coverage
- [ ] Identified all new dependencies and assessed each
- [ ] Recorded at least one Positive Note
- [ ] Confirmed: every Critical item has a location reference and a specific description
