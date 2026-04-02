# UAT Criteria Guide

How to write human-executable UAT scenarios from acceptance criteria, the distinction between AC and UAT, and 5 complete translation examples.

---

## The Distinction Between AC and UAT

An **acceptance criterion (AC)** is a testable statement of required system behavior. It is typically expressed in Given/When/Then form. ACs can be verified by automated tests, by code inspection, or by a human.

A **UAT criterion** is a human-executable procedure: given a state the tester can establish without code access, a user action they perform, and an observable outcome they verify. UAT criteria require no knowledge of the implementation.

The same behavior can be expressed as an AC and as a UAT criterion, but they serve different purposes:

| | Acceptance Criterion | UAT Criterion |
|---|---|---|
| **Audience** | Developer writing tests | Human executing tests |
| **Expressed as** | Testable behavior statement | Step-by-step procedure |
| **Verified by** | Automated test or code inspection | Human observation |
| **Requires code knowledge** | Yes | No |
| **Artifact** | SPEC.md | QA.md |

---

## The UAT Criterion Format

```
### UAT-[AC-NN-MM]: <short name>

**Source:** AC-NN-MM — <original acceptance criterion text>
**Given:** <the starting state a human tester can establish>
**Action:** <what the tester does — a user action>
**Expected outcome:** <what the tester observes>
**Pass signal:** <the unambiguous indicator that this criterion is met>
```

**Given:** Describe the precondition in terms the tester can establish without code. "A user account exists with email test@example.com and password TestPass123" is valid. "The `User` object has `is_verified=True`" is not — that requires code access.

**Action:** Describe what the tester does. Use the vocabulary of the interface (UI: "click the Login button", CLI: "run `timecli start 'writing'`", API: "send a POST request to /api/login with the JSON body...").

**Expected outcome:** Describe what the tester observes. Be specific and measurable. "The dashboard appears" is ambiguous. "The page title reads 'Dashboard' and the user's name appears in the top right corner" is specific.

**Pass signal:** Define the binary pass/fail indicator. This should be so unambiguous that two different testers would agree on the result. If the pass signal requires interpretation, make it more specific.

---

## 5 Complete AC-to-UAT Translation Examples

### Example 1: Simple login flow

**Original AC:**
> AC-01-01: Given a registered user with email and password, when they submit valid credentials, the system authenticates them and redirects to the dashboard.

**UAT Criterion:**

```
### UAT-AC-01-01: Login with valid credentials

**Source:** AC-01-01 — Registered user with valid credentials is authenticated and redirected to dashboard.
**Given:** A user account exists with email "tester@example.com" and password "Verify2025!".
  The tester is on the login page (not logged in).
**Action:** Enter "tester@example.com" in the Email field. Enter "Verify2025!" in the
  Password field. Click "Sign In".
**Expected outcome:** The browser navigates to /dashboard. The page heading reads "Dashboard".
  The top-right corner displays "tester@example.com" or the user's name.
**Pass signal:** URL contains /dashboard AND user identifier is visible in the header.
```

### Example 2: Invalid input rejection

**Original AC:**
> AC-01-03: Given a login attempt with an unrecognized email, the system returns an error and does not reveal whether the email or password was incorrect.

**UAT Criterion:**

```
### UAT-AC-01-03: Login failure does not reveal which field is wrong

**Source:** AC-01-03 — Login with unrecognized email shows generic error, does not
  indicate which field failed.
**Given:** The tester is on the login page. The email "notregistered@example.com"
  does not exist in the system.
**Action:** Enter "notregistered@example.com" in the Email field. Enter any value in
  the Password field. Click "Sign In".
**Expected outcome:** An error message appears. The message does not contain the words
  "email not found", "no account", "incorrect email", or any phrase that identifies
  the email as the failing field. The message text is a generic failure message such
  as "Invalid credentials" or "Email or password is incorrect."
**Pass signal:** Error message is visible AND error message text does not identify
  the email field as the cause of failure.
```

### Example 3: CLI command with file output

**Original AC:**
> AC-03-02: Given a stopped timer, when the user runs `timecli export`, a CSV file is created at the current working directory named `timelog-YYYY-MM-DD.csv` containing all recorded sessions.

**UAT Criterion:**

```
### UAT-AC-03-02: Export timer log to CSV

**Source:** AC-03-02 — Running `timecli export` creates a dated CSV file in the
  current directory with all recorded sessions.
**Given:** At least 2 timer sessions have been recorded (started and stopped) on the
  current day. No timer is currently running. The tester is in a terminal at any
  working directory.
**Action:** Run `timecli export`.
**Expected outcome:** The terminal prints a confirmation message. A file named
  `timelog-<today's date in YYYY-MM-DD format>.csv` appears in the current
  working directory. Opening the file in a text editor or running `cat` on it
  shows a header row and one data row per recorded session.
**Pass signal:** File exists with today's date in its name AND file contains a
  header row AND file contains at least 2 data rows matching the recorded sessions.
```

### Example 4: State change observable after action

**Original AC:**
> AC-04-05: Given an admin user, when they deactivate a user account, the deactivated user can no longer log in.

**UAT Criterion (two parts — one per observable outcome):**

```
### UAT-AC-04-05-A: Admin can deactivate a user account

**Source:** AC-04-05 — Admin can deactivate a user; deactivated user cannot log in.
**Given:** The tester is logged in as an admin user. A second user account exists with
  email "victim@example.com".
**Action:** Navigate to Admin → Users. Locate "victim@example.com" in the user list.
  Click "Deactivate". Confirm the deactivation in the dialog.
**Expected outcome:** The user's status in the list changes to "Inactive". A success
  notification appears.
**Pass signal:** "victim@example.com" shows "Inactive" status in the admin user list.

### UAT-AC-04-05-B: Deactivated user cannot log in

**Source:** AC-04-05 — Admin can deactivate a user; deactivated user cannot log in.
**Given:** The "victim@example.com" account has been deactivated (see UAT-AC-04-05-A).
  The tester is logged out.
**Action:** Navigate to the login page. Enter "victim@example.com" and the account's
  password. Click "Sign In".
**Expected outcome:** The login fails. An error message appears. The tester is not
  redirected to the dashboard.
**Pass signal:** Login page remains visible AND error message is displayed AND
  URL does not contain /dashboard.
```

### Example 5: Time-bounded or threshold behavior

**Original AC:**
> AC-05-01: Given a payment submission, the system processes the payment and returns a result within 10 seconds under normal load.

**UAT Criterion:**

```
### UAT-AC-05-01: Payment processing completes within 10 seconds

**Source:** AC-05-01 — Payment processed and result returned within 10 seconds
  under normal load.
**Given:** The tester has a product in their cart. The tester is on the checkout page
  with a valid test payment card entered (use the designated test card number from
  the test environment documentation).
**Action:** Note the current time. Click "Place Order".
**Expected outcome:** A confirmation page or success message appears. The time elapsed
  from clicking "Place Order" to the confirmation page loading is 10 seconds or less.
**Pass signal:** Order confirmation visible AND elapsed time ≤ 10 seconds (tester
  verifies by observation; for a stricter check, the tester records start time
  and page load time).
```

---

## Common UAT Translation Mistakes

**Criterion requires reading code or logs to verify.** If the expected outcome is "verify the database record has `status='processed'"`, this is not human-executable UAT. Translate to an observable outcome in the interface: "the order status on the Order History page reads 'Processed'."

**Criterion is ambiguous about the starting state.** "A user is logged in" is insufficient if there are multiple user roles with different access. Specify: "A user logged in with the 'standard' role (not admin)."

**Pass signal is subjective.** "The page looks correct" is not a pass signal. "The page heading reads 'Order Confirmed' and an order number in the format ORD-XXXXXX is visible" is a pass signal.

**Multiple independent behaviors in one criterion.** If the pass signal requires two unrelated things to be true, split the criterion. UAT criteria should verify one behavior.

**No source reference.** Every UAT criterion must reference its AC-NN-MM source. A criterion with no source is untraceable and cannot be verified against requirements.
