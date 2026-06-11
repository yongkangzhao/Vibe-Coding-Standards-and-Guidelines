# Standard 10: The Required Development Loop — Spec → Interface → Tests → Implementation

> A human engineer might skip the spec and rely on domain knowledge accumulated over months. An agent starts fresh every session — it has no internalized context. Without spec, interface, and tests already in place, the agent designs on the fly, and its design reflects local context rather than the global architecture decisions that have already been made. The development loop exists to ensure that by the time an agent writes implementation code, three layers of constraints already define what "correct" looks like.

> **Standards** (must follow):
> - Follow the sequence: Spec, Interface, Tests, Implementation — in that order
> - Tests must fail before implementation begins (TDD red phase confirmation)
> - Interfaces must exist in the shared contracts file before any service code is written
>
> **Guidelines** (recommended):
> - Exact spec format (README structure, level of detail)
> - Test framework and fixture conventions
> - How to handle iterative refinement within the loop

Standards 3–9 describe the structure. This is the process — the order of operations for every new module or feature:

```
1. Spec        — README: what does this module own? what does it NOT own?
2. Interface   — method signatures in interfaces.py, before any implementation
3. Tests       — write against the interface; confirm they FAIL (red phase)
4. Implementation — make tests pass, within the constraints already set
5. Verify      — drive the real path; observe it working in the deployed environment
```

By the time you write implementation code, three layers of constraints already exist. Implementation's only job is to satisfy them. And after implementation, one gate remains: confirming the thing actually runs through its real path — because "tests green" is not the last word.

**Spec first.** The README isn't documentation written after the fact — it's the design decision that happens before any code. It answers "what belongs here?" and crucially, "what does NOT belong here?" This is where the commission rate debate gets resolved. Once it's in the spec, the interface encodes it, and the question is closed.

**Interface second.** Not the implementation — just the shape:

```python
# common/interfaces.py — written before service.py exists

class NotificationServiceInterface(Protocol):
    def notify_assignment(self, entity_id: str, user_id: str) -> None: ...
    def notify_completion(self, entity_id: str) -> None: ...
```

This locks scope (if it's not in the interface, it's not part of this module), enables parallel agent work (other modules build against this before the implementation exists), and gives the implementer agent a concrete target.

**Tests third — and they must fail.** Write tests against the interface before implementation exists. Every test should fail (stubs raise `NotImplementedError`). If tests pass before implementation, the tests are wrong. Confirm the red state — it's not a formality, it's proof that the tests describe real behavior.

**Implementation last.** The agent reads the interface, reads the failing tests, and writes code until they pass. There is no design work left. There is only build work. This is the constraint that keeps agents focused and prevents them from inventing architecture mid-implementation.

**Verify through the real path — "tests green" is not the last gate.** Passing tests prove the units behave as their seams describe; they do not prove the wired-up system serves a real request. The loop is not done until the change has been exercised through its actual path and *observed working in the deployed environment*. Two failure modes hide precisely in the gap that green tests leave open:

- **Unit tests certify the seams, not the wiring between them.** A handler that reads `current_user.uid` when the auth model field is `user_id` raises `AttributeError` and 500s every caller — yet every unit test stays green, because they mock the network or call the service directly and never run the three lines of router glue. An end-to-end assertion must drive the **real route** (real router, real dependency, real auth model), not a mocked stand-in for it.
- **An empty page is not proof of correctness.** A failed fetch can render as an empty-state, a `null` can render as `$0.00`, an optimistic update can show a phantom success — the screen looks fine while the truth is broken. So for any UI change, the closing evidence is a **visual proof**: render the actual route at the reviewed viewport, against staging-like data, with the change applied, and capture the screenshot *after* the page-specific content is in the DOM — not a passing test, not a code reading, not an HTML mock.

Treat this as the final, non-skippable step of the loop: a real-route/e2e assertion plus a genuine observation (a journey test green against the wired system; a screenshot of the real screen for UI work). When the deployed app misbehaves, also resist blaming the deploy reflexively — `git diff` the working tree first; most often the deploy shipped exactly what's committed, and the committed code is the bug.

The critical point for AI agents specifically: a human engineer might skip the spec and rely on domain knowledge. An agent starts fresh every session — it has no internalized context. Without spec, interface, and tests already in place, the agent designs on the fly, and its design reflects local context, not the global architecture decisions that have already been made. And an agent will report "all tests pass" as "done" unless the loop forces it to look at the real thing running — because to the agent, a green test suite is indistinguishable from a working feature until something makes it observe otherwise.

**Design is a human activity. Implementation is what agents do.**
