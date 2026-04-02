# Uncertainty Patterns

Reference for the `confidence-gate` skill. Covers the four uncertainty categories, per-category detection signals and resolution strategies, and the confidence scoring worksheet.

---

## The four uncertainty categories

### 1. Requirements uncertainty

What is being asked for is unclear.

**Subcategories:**

- **Missing context** — the request omits information needed to choose between valid approaches. Example: "add a search feature" without specifying whether full-text, fuzzy, or exact-match is expected.
- **Conflicting signals** — the stated goal contradicts something else in the conversation or codebase. Example: "keep it simple" alongside a requirement that implies significant complexity.
- **Implicit assumptions** — the request relies on shared understanding that has not been stated. Example: "update the config" assumes a specific config file is the target when multiple exist.

**Detection signals:**

- Vague verbs: improve, clean up, refactor, optimize, fix (without describing the broken behavior)
- Multiple valid interpretations with meaningfully different outputs
- A request that contradicts a constraint visible in the codebase or previous conversation
- Absence of an acceptance criterion — no way to know when done

**Impact rating:** High. Requirements uncertainty affects what gets built, not just how. Proceeding under requirements uncertainty produces technically correct output that solves the wrong problem.

**Resolution strategies:**

- Restate the task as a specific, falsifiable objective and ask for confirmation
- Ask: "What would done look like?" or "What is the output you'll use this for?"
- Surface conflicting signals explicitly: "This seems to conflict with X — which takes precedence?"
- Identify the single most load-bearing assumption and verify it before anything else

---

### 2. Technical uncertainty

How to implement is unclear.

**Subcategories:**

- **Unknown APIs or systems** — the task involves a library, service, or internal system that has not been verified in this session. Example: calling an internal API endpoint whose contract has not been confirmed.
- **Unverified compatibility** — the approach works in the general case but may not work in this environment. Example: a pattern that requires a minimum framework version not confirmed in the project.
- **Untested patterns** — the implementation approach is novel in this codebase and carries integration risk. Example: introducing an async queue in a codebase that has no prior async patterns.

**Detection signals:**

- Task requires knowledge of a specific file, endpoint, schema, or API that has not been read or confirmed
- The approach depends on a version, runtime, or platform assumption
- No prior art in the codebase for the pattern being introduced
- The task involves a third-party system whose behavior under edge cases is not known

**Impact rating:** Medium to high. Technical uncertainty typically requires revision rather than a full restart, but can cascade into scope creep if the wrong approach is chosen early.

**Resolution strategies:**

- Read the relevant file, schema, or API contract before planning
- Check for existing patterns in the codebase that can be extended rather than replaced
- Ask: "Is there a working example of this pattern elsewhere in the project?"
- When environment facts are uncertain, check (package.json, pyproject.toml, Dockerfile) rather than assume

---

### 3. Scope uncertainty

What is in or out of bounds is unclear.

**Subcategories:**

- **No explicit boundary** — the task description does not specify what is adjacent but excluded. Example: "update the authentication flow" — does this include the session management layer?
- **Scope that grew mid-task** — an earlier decision or discovery expanded what "done" means, but the boundary was never re-confirmed. Example: finding a related bug while implementing a feature.
- **Multiple valid interpretations** — the task can be read narrowly (touch one file) or broadly (touch a whole subsystem), and both are defensible.

**Detection signals:**

- Task description uses collective nouns without enumeration ("the handlers", "the tests", "the config")
- A discovery during execution reveals that a stated scope is insufficient to actually solve the problem
- No explicit list of what is not included
- The natural completion of the task would touch files or systems not mentioned in the request

**Impact rating:** Medium. Scope uncertainty rarely blocks a start but frequently causes rework at the end. Its danger is that it surfaces late — after significant work is already done.

**Resolution strategies:**

- Before starting, enumerate the files or systems that will be touched and confirm
- When scope grows mid-task, surface it explicitly: "I've found that X also needs to change — is that in scope?"
- Ask: "What is explicitly not in scope?" — the answer often clarifies the boundary faster than asking what is in scope
- Prefer narrow scope with explicit expansion over broad scope with implicit exclusions

---

### 4. Assumption uncertainty

Things being taken for granted that may be wrong.

**Subcategories:**

- **Environmental assumptions** — facts about the runtime, filesystem, or infrastructure that have not been verified. Example: assuming a specific environment variable is set, or that a directory exists at a given path.
- **User knowledge assumptions** — assuming the user has context that was not stated in the conversation. Example: referencing a design decision made in a previous session without confirming it is still active.
- **Data shape assumptions** — assuming a specific structure for data that has not been confirmed. Example: assuming a JSON response has a particular field without reading the schema or a live example.

**Detection signals:**

- The plan relies on a path, variable, or configuration value that has not been confirmed in this session
- The response references "as we discussed" or "as established" when no such establishment is visible in the current context
- The implementation handles a data structure that has not been read from the actual source
- The approach assumes a user workflow or mental model that was inferred rather than stated

**Impact rating:** Low to high depending on centrality. Environmental assumptions about widely-used paths are usually low-impact (easily corrected). Data shape assumptions about a core entity are high-impact (cascade into every layer that touches that entity).

**Resolution strategies:**

- Read before assuming: check the actual file, schema, or config rather than inferring from naming conventions
- When referencing a prior decision, restate it and ask for confirmation: "I'm assuming X is still the approach — is that right?"
- For data shapes, read a concrete example (a fixture, a live API response, a migration file) rather than inferring from field names
- Surface environmental assumptions in the confidence block so the user can correct them before execution begins

---

## Confidence scoring worksheet

### Step 1: Enumerate uncertainties

List every uncertainty identified across all four categories. Be specific — "unclear requirements" is not a list item; "unclear whether the search should be case-sensitive" is.

### Step 2: Assign impact weights

For each uncertainty, assign an impact weight:

| Impact level | Definition | Points subtracted |
|---|---|---|
| High | Would change the approach, the target file, or the meaning of the output if wrong | 15 |
| Medium | Would require revision but not a full restart | 8 |
| Low | Cosmetic or easily corrected after the fact | 3 |

### Step 3: Calculate score

Start at 100. Subtract the points for each uncertainty. Round to the nearest 5%.

**Example:**

- "Unclear whether this should update the v1 or v2 API handler" → High → −15
- "Not sure if the existing tests cover this path" → Medium → −8
- "Unsure whether to use single or double quotes in this codebase" → Low → −3

Score: 100 − 15 − 8 − 3 = 74% → round to 75%

### Step 4: Apply threshold guidance

| Score | Action |
|---|---|
| 85% or above | Proceed. Document carried uncertainties as named assumptions inline. |
| 70–84% | Ask one to two targeted questions. Choose the questions that address the highest-impact uncertainties. Proceed after answers received. |
| Below 70% | Stop. Restate understanding of the task in plain language. Ask for confirmation before any further action. Do not proceed with qualifications or hedges — halt. |

### Notes on scoring discipline

- Do not average impact across all uncertainties. One high-impact unknown is disqualifying regardless of how many low-impact items are resolved.
- Do not inflate the score because the task "seems easy." Confidence measures clarity, not capability.
- At exactly 85%, proceed — the threshold is inclusive.
- When two uncertainties are correlated (resolving one resolves the other), count them as one item at the higher impact weight.
- When an uncertainty cannot be resolved by asking a question (e.g., it requires reading a file), read the file before scoring — do not assign impact weight to uncertainties that are resolvable by immediately available actions.

---

## Quick reference: category-to-question mapping

| Category | First question to ask |
|---|---|
| Requirements | "What would done look like — what output or behavior will you check?" |
| Technical | "Is there an existing example of this pattern in the codebase I should follow?" |
| Scope | "What is explicitly not in scope for this change?" |
| Assumptions | "I'm assuming X — is that still current?" |

When confidence is between 70% and 84%, select the one or two questions from this table that correspond to the highest-impact uncertainties in the scored list. Do not ask all four unless all four categories have high-impact open items.
