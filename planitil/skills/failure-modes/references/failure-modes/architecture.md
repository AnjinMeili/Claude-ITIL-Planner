# Architecture Failure Modes

Eight named failure modes for architectural decisions — technology choices, service boundaries, data models, and system structure. Each failure mode has a symptom pattern and an explicit STOP condition. A STOP condition is a specific, observable signal — if present, halt and restart with the condition resolved.

---

## 1. Resume-Driven Design

**Symptom:** Technology is chosen for novelty, familiarity, or resume value rather than fit for the problem. The decision process optimizes for what is interesting to work with rather than what is correct for the constraints.

**STOP condition:** No explicit "why not simpler" comparison exists in the decision record — no documented consideration of whether a lower-complexity alternative would meet the requirements.

---

## 2. Abstraction Anticipation

**Symptom:** Abstractions are built for requirements that do not exist yet. The system is made generic and extensible for future consumers that have not been identified.

**STOP condition:** Any abstraction, interface, or extension point has only one current consumer — it cannot be pointed to two distinct current callers that justify its generality.

---

## 3. Coupling Camouflage

**Symptom:** Tight coupling is hidden behind thin interfaces. The interface appears to decouple components, but its method signatures or semantics only make sense for one concrete implementation.

**STOP condition:** Any interface contains methods whose names, parameters, or semantics only make sense for one specific concrete implementation — a second implementation could not be written that satisfies the interface without distortion.

---

## 4. Boundary Erosion

**Symptom:** Layer boundaries are violated for convenience. A component reaches directly into another layer, bypassing the declared architecture, creating hidden dependencies that the architecture diagram does not show.

**STOP condition:** Any component has a direct dependency on another component in a layer it is not supposed to interact with, bypassing the declared boundary or intermediary in the architecture.

---

## 5. Scaling Speculation

**Symptom:** The architecture is optimized for a scale or load that has not been observed and is not backed by data. Complexity is added to handle hypothetical future load.

**STOP condition:** Any scaling argument in the decision record — for caching, sharding, replication, queue depth, or similar — is not backed by measured load data or a credible projection with a named source.

---

## 6. Testing Afterthought

**Symptom:** The architecture makes testing hard. Components are structured in ways that require real external services, network access, or complex environment setup to unit test.

**STOP condition:** Any component requires a real external service, live database, or real network connection to execute its unit tests — it cannot be tested in isolation with fakes or stubs.

---

## 7. Reversibility Gap

**Symptom:** The architecture includes decisions that are extremely hard to undo — vendor lock-in, proprietary protocols, or data formats with no migration path. These decisions are treated as normal design choices rather than as commitments requiring explicit documentation.

**STOP condition:** Any decision that locks in a vendor, proprietary protocol, or data format has no exit path documented — no described migration path or abstraction layer that would allow the dependency to be replaced.

---

## 8. Consistency Cosmetics

**Symptom:** Patterns are applied for consistency rather than fit. DDD, CQRS, event sourcing, hexagonal architecture, or similar patterns are applied to domains that do not have the complexity or behavior that justifies them.

**STOP condition:** Any instance of DDD aggregates, CQRS command/query separation, event sourcing, or similar pattern is applied to a domain that does not have the complexity — specifically, a domain without multiple bounded contexts, complex domain rules, or audit/temporal requirements — that the pattern exists to address.
