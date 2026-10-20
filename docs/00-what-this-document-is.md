# What This Guide Is—and Is Not

AI coding tools reduce the cost of producing a plausible implementation. They do
not reduce the need to decide ownership, protect data, verify integrations, or know
which artifact is running. In practice, lowering the cost of writing code moves the
bottleneck toward specification, review, and evidence.

This guide describes an engineering system for that new bottleneck.

## The scope

The intended audience is building software that must be maintained: a product with
users, state, integrations, multiple contributors, or meaningful operational risk.
For a disposable experiment, many of these controls are unnecessary. If the
experiment becomes a product, adopt the controls before its accidental architecture
hardens into dependencies and data.

The standards cover:

1. Domain ownership and modular architecture.
2. Specifications and typed contracts.
3. Test-first behavior changes and realistic integration evidence.
4. Independent, risk-calibrated review.
5. CI selection, deployment identity, preview lifecycle, and cleanup.
6. Data retention, erasure, concurrency, and external effects.
7. Gradual modernization of large codebases.

## Why agent-assisted codebases fail differently

The underlying defects are familiar: duplicated behavior, hidden coupling, missing
authorization, nondeterministic tests, stale deployments, and undocumented state.
Agents change their rate and shape:

- A fresh session has little institutional memory unless the repository encodes it.
- Locally plausible choices can be produced across a large surface very quickly.
- The same ambiguous question may receive a different answer in each session.
- Green mocked tests can encourage confident claims about paths never exercised.
- Review capacity, not typing speed, becomes the limiting resource.

The response is not more elaborate prompting. It is to make important decisions
discoverable and enforceable: ownership documents, contracts, dependency checks,
behavioral tests, immutable revisions, and observable release state.

## Standards, guidelines, and evidence

This guide uses three levels deliberately:

- **Standards** define properties that must remain true, such as authorization
  being server-side or a deployment being bound to an exact artifact.
- **Guidelines** offer useful defaults that may vary by stack, such as a directory
  name or a suggested file-size threshold.
- **Evidence** proves a standard for a particular revision: a failing-then-passing
  test, an architecture check, a reviewed diff, a live probe, or a screenshot.

Documentation alone is not enforcement. A repeated or objectively detectable
violation should become a linter, test, hook, or CI gate with an actionable error.

## What this guide is not

- **Not a prompt catalog.** Prompting can improve an interaction; it cannot supply
  missing architecture or release evidence.
- **Not tied to one agent or stack.** Examples are illustrative, not requirements.
- **Not a mandate for maximum process.** Controls should be proportional to risk.
- **Not a case for premature microservices.** Distribution adds operational and
  consistency costs; it should follow measured need.
- **Not a claim that every change needs the full suite.** Narrow CI is valuable when
  an automated, fail-closed classifier can prove the change is narrow.
- **Not a substitute for legal, security, accessibility, or domain expertise.**
- **Not complete forever.** Repeated failures should evolve the standards and their
  automated enforcement.

## The core idea

The most useful repository is one in which a fresh contributor can make a correct
change without relying on private memory. Vibe coding becomes sustainable when
architecture, expected behavior, and delivery state are properties of the system,
not facts held by the person who happened to be present last week.
