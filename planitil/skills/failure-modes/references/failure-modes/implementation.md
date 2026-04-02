# Implementation Failure Modes

Eight named failure modes for code generation and implementation work. Each failure mode has a symptom pattern and an explicit STOP condition. A STOP condition is a specific, observable signal — if present, halt and restart with the condition resolved.

---

## 1. Interface Drift

**Symptom:** The implementation diverges from the agreed interface. Function signatures, return types, error shapes, or field names differ from what callers expect, discovered at integration time.

**STOP condition:** Any function, method, or API endpoint has a signature — parameter names, types, order, or return shape — that does not match what the declared caller or contract specifies.

---

## 2. Error Swallowing

**Symptom:** Errors are caught and silently discarded. The calling code receives a success signal or null when an error actually occurred, making failures invisible until they surface as corrupted state downstream.

**STOP condition:** Any `catch`, `except`, `rescue`, or equivalent error-handling block does not re-raise the error, log it, or return an explicit error signal to the caller — the error exits the block with no observable trace.

---

## 3. Implicit Dependency

**Symptom:** Code requires environmental state that is not declared in its interface. The function works in one environment and fails silently in another because its inputs are not explicit.

**STOP condition:** Any function accesses a global variable, reads an environment variable, or calls a singleton not declared in its function signature or constructor — the dependency is real but invisible to callers.

---

## 4. Test-Last Trap

**Symptom:** Implementation is written without testability in mind. Functions are large, monolithic, and tangled with their dependencies, making after-the-fact tests require significant refactoring or end-to-end setup.

**STOP condition:** Any function exceeds 30 lines and has no clear seam — a point where a dependency could be injected or behavior could be substituted — making it untestable without real infrastructure.

---

## 5. Naming Deception

**Symptom:** Names imply different behavior than what is implemented. Callers form incorrect mental models from the name and use the function incorrectly or write tests against the wrong assumptions.

**STOP condition:** Any function named with a `get*` or `fetch*` prefix has side effects, or any function named with an `is*` or `has*` prefix returns a non-boolean value.

---

## 6. Premature Optimization

**Symptom:** Complexity is added for performance without measurement. Caches, pools, lazy evaluations, and batching are introduced based on anticipated load rather than observed behavior.

**STOP condition:** Any caching, connection pooling, lazy evaluation, memoization, or batching mechanism is added without a profiling baseline that demonstrates the performance problem it solves.

---

## 7. Defensive Paranoia

**Symptom:** Error handling is added for states that cannot occur. Null checks, guard clauses, and fallback branches handle conditions already ruled out by the type system, surrounding code, or invariants — adding noise without safety.

**STOP condition:** Any null check, undefined guard, or defensive branch operates on a value that the type system, the call site, or an established invariant already guarantees is non-null and valid — the guard is logically unreachable.

---

## 8. Scope Escape

**Symptom:** The implementation adds features, behaviors, or code paths beyond what was specified. Extra convenience methods, unasked-for configuration options, and "while I'm here" additions appear in the diff.

**STOP condition:** Any code path, function, parameter, or branch is not traceable to a requirement, acceptance criterion, or task in `SCOPE.md` — it exists because it seemed useful, not because it was asked for.
