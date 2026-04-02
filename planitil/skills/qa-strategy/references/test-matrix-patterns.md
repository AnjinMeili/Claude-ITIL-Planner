# Test Matrix Patterns Reference

How to populate the test matrix: candidate identification, tool selection, and coverage threshold guidance.

---

## Unit Test Candidates

Unit tests verify a single function or class in isolation, with all dependencies replaced by test doubles. They run fast, are easy to write, and give precise failure signals.

**Strong candidates for unit tests:**

- Pure functions with no side effects (business logic, calculations, transformations, validations)
- Functions with complex branching logic (every branch is a candidate for a distinct test case)
- Functions that handle edge cases (empty input, zero, null, maximum values, unicode)
- Domain model methods (e.g., `Order.total_price()`, `Session.is_expired()`, `User.has_permission()`)
- Parsers and serializers
- Error classification logic (which exception type produces which error response)

**Poor candidates for unit tests (use integration or E2E instead):**

- Functions whose primary effect is a database write — you cannot meaningfully test the write without the database
- Functions that orchestrate other functions without adding logic of their own
- Framework boilerplate (routes, middleware registration, ORM model definitions)

### Coverage Threshold Guidance for Unit Tests

**80% line coverage** on business logic modules is a realistic production default.

The 80% figure is not a magic number — it represents the point where increasing coverage starts requiring tests for paths that are either untestable (unreachable error paths in framework code) or trivially obvious (a one-line getter). Adjust based on:

- Risk: Higher coverage targets are justified for payment, authentication, and data transformation modules.
- Team maturity: A team new to testing often benefits from starting lower (60%) and raising the bar as test hygiene improves.
- Legacy code: Setting a high threshold on a legacy codebase with no tests produces a massive one-time cost. Ratchet: enforce the current coverage level; only allow it to increase.

**100% coverage is almost always wrong.** It forces tests on trivially correct code, creates maintenance burden, and incentivizes writing tests purely to hit the number.

---

## Integration Test Candidates

Integration tests verify that two components interact correctly across a real boundary. They are slower than unit tests and require the real dependency (or a faithful substitute like a test database).

**Strong candidates for integration tests:**

- Database interactions: Does the ORM produce the correct SQL? Does the query return the expected records? Does a write transaction commit correctly and roll back on error?
- External API clients: Does the client construct the correct request? Does it parse the response correctly? Does it handle error responses (4xx, 5xx, timeout)?
- File I/O: Does the parser read the format correctly? Does the writer produce a valid file? Does it handle missing files and permission errors?
- Message queue interactions: Does the producer serialize messages correctly? Does the consumer deserialize and process them correctly?
- Cache interactions: Does the cache hit return the correct value? Does the miss correctly populate the cache?

**Coverage metric for integration tests:**

Percentage coverage is the wrong metric for integration tests. The right question is: are all meaningful interaction scenarios covered?

For a database module:
- Happy path: correct data returned on a valid query
- Not found: correct behavior when the record does not exist
- Constraint violation: correct error raised on a duplicate insert
- Connection failure: correct behavior when the database is unavailable

If all four scenarios are covered, the integration coverage is complete — regardless of what percentage of lines that represents.

---

## E2E Test Candidates

E2E tests verify a complete user-facing workflow from start to finish, exercising all layers of the stack. They are slow, brittle, and expensive to maintain. Use them selectively.

**Strong candidates for E2E tests:**

- The core value proposition of the application (the primary workflow a user comes to the product to execute)
- Authentication and session management (the flow that gates everything else)
- Payment or financial transactions (high-value, high-risk workflows where a silent failure is unacceptable)
- Data import/export (workflows that cross system boundaries with user-supplied data)

**Poor candidates for E2E tests:**

- Any workflow already covered by unit and integration tests at a level of confidence that makes E2E redundant
- UI validation logic (unit test this — E2E tests for validation are brittle)
- Error paths that are difficult to trigger reliably in an E2E context

### E2E Coverage Strategy

Do not target percentage coverage for E2E. Target: every workflow in the "must never break" category in QA.md's regression scenarios has an automated E2E check.

E2E tests on critical paths only. If you find yourself writing E2E tests for edge cases, move the logic to the unit or integration layer and test it there.

---

## Tool Selection Guide

### Python

| Type | Tool | Notes |
|---|---|---|
| Unit | pytest | Standard. Fixtures, parametrize, and plugin ecosystem. |
| Unit mocking | pytest-mock / unittest.mock | Use pytest-mock for fixture-style mocking. |
| Integration (DB) | pytest + sqlalchemy / pytest-django | Use a real test database, not SQLite if production is Postgres. |
| Integration (HTTP) | responses / httpretty | Mock HTTP at the transport layer, not the function level. |
| E2E (CLI) | subprocess + assertions in pytest | Run the CLI as a subprocess; assert stdout, exit code, and side effects. |
| E2E (web) | playwright | Headless browser. Reliable and fast. |
| Coverage | pytest-cov | Generates line and branch coverage reports. |

### TypeScript/JavaScript

| Type | Tool | Notes |
|---|---|---|
| Unit | jest or vitest | vitest is faster for Vite-based projects. |
| Integration | jest + supertest (API) | supertest runs the Express/Fastify app without a real server. |
| E2E | playwright | Cross-browser. First choice for web E2E. |
| Coverage | istanbul (built into jest/vitest) | V8 provider is faster; babel provider is more accurate for branches. |

### Go

| Type | Tool | Notes |
|---|---|---|
| Unit | go test (built-in) | No external framework needed for most cases. |
| Integration | go test + testcontainers-go | Spin up real dependencies (Postgres, Redis) in Docker. |
| E2E | go test + httptest | httptest.NewServer for API E2E. |
| Coverage | go test -cover | Built in. Target by package, not global. |

---

## Test Organization Conventions

**Co-locate tests with the code they test.** `user_service.py` and `user_service_test.py` in the same directory. Do not use a separate top-level `tests/` directory unless the language convention mandates it (Go uses `_test.go` in the same package; follow the convention).

**Name test functions to describe behavior, not implementation.** `test_returns_error_when_user_not_found` not `test_get_user_2`. Test function names are the documentation of what the function is supposed to do.

**One assertion per test is a guideline, not a rule.** Group assertions that together verify a single behavior. Separate tests that verify distinct behaviors.

**Fixtures over setup/teardown.** Use test fixtures or factory functions to set up test state. Avoid complex setUp/tearDown methods that obscure what state each test requires.
