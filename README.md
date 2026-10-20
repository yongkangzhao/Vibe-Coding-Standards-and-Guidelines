# Production-Grade Vibe Coding

> AI agents make code generation cheap. These standards keep understanding,
> verification, and change control from becoming the bottleneck.

“Vibe coding” can mean anything from prompting a prototype into existence to
running AI agents inside a mature engineering system. This guide is for the
second case: software expected to survive growth, handoffs, incidents, changing
requirements, and more than one contributor.

The central premise is simple:

> Agents amplify the structure that already exists. Clear boundaries, executable
> contracts, fast feedback, and observable delivery compound into leverage.
> Ambiguity, hidden coupling, flaky tests, and manual release state compound into
> rework.

This is a generic guide. Its examples use fictional domains and common technology
patterns; it intentionally contains no private project details.

## Start here

If you are adopting these practices incrementally, begin with five moves:

1. Give every domain an explicit owner and “does not own” boundary.
2. Put behavior behind typed contracts and enforce dependency direction.
3. Write a failing behavioral test before changing behavior.
4. Verify critical journeys through the real integration path.
5. Make CI and releases revision-bound, observable, and proportional to risk.

The goal is not maximum ceremony. The goal is the smallest system that produces
trustworthy evidence for the risk of the change.

## Reading path

### 1. Why structure matters

- [What this guide is—and is not](docs/00-what-this-document-is.md)
- [What breaks without durable engineering constraints](docs/02-what-breaks.md)

### 2. Architecture and ownership

- [Standard 1: Use a modular monolith as the default](docs/01-modular-monolith.md)
- [Standard 2: Constraints define where and how](docs/03-constraints.md)
- [Standards 3–9: Structure, specifications, contracts, and guardrails](docs/04-structural-guidelines.md)
- [Standard 13: Design for change without premature distribution](docs/07-extractability.md)

### 3. Building and reviewing changes

- [Standard 10: Spec → Contract → Tests → Implementation → Proof](docs/05-development-loop.md)
- [Standards 11–12: Human judgment and the rule flywheel](docs/06-human-standards.md)
- [Standard 14: Coordinated agent work](docs/08-agent-teams.md)

### 4. Verification and delivery at scale

- [Standard 15: Layered, deterministic test infrastructure](docs/09-test-infrastructure.md)
- [Standard 16: Data lifecycle and state safety](docs/10-data-retention.md)
- [Standards 17–19: Risk-based CI, code-health ratchets, and release state](docs/11-scalable-delivery.md)

### 5. Adoption

- [Implementation guide](docs/12-implementation-guide.md)

### Reference

- [Appendix A: Common mistakes that become rules](docs/appendix/A-common-mistakes.md)
- [Appendix B: Guardrail and hook taxonomy](docs/appendix/B-hook-taxonomy.md)

## The operating model

The guide separates four ideas that are often mixed together:

| Layer | Purpose | Typical evidence |
|---|---|---|
| Architecture | Decide ownership and dependency direction | module specifications, typed contracts, boundary checks |
| Development | Change behavior in a controlled sequence | RED test, focused GREEN checks, reviewed diff |
| Verification | Prove integrated user-visible behavior | real-engine integration tests, golden journeys, screenshots or API evidence |
| Delivery | Prove the intended artifact is serving and clean up safely | immutable revision, rollout health, live probes, verified teardown |

Fast teams do not collapse these layers into “the tests passed.” They make each
claim explicit and collect only the evidence needed to support it.

## Reusable agent skills

The repository also packages the standards as eleven self-contained skills under
[`.skills/`](.skills/). The prose explains why; the skills turn it into repeatable
agent behavior.

### Architecture and workflow

| Skill | Purpose |
|---|---|
| [`architecture-and-module-boundaries`](.skills/architecture-and-module-boundaries/SKILL.md) | Domain ownership, dependency direction, structural boundaries, and evolutionary architecture |
| [`interface-first-development`](.skills/interface-first-development/SKILL.md) | Specifications and contracts before implementation |
| [`tests-as-ground-truth`](.skills/tests-as-ground-truth/SKILL.md) | Behavioral tests, realistic infrastructure, and deterministic fixtures |
| [`agent-team-review-loop`](.skills/agent-team-review-loop/SKILL.md) | Bounded assignments, independent review, and efficient repair loops |
| [`guardrails-and-rule-flywheel`](.skills/guardrails-and-rule-flywheel/SKILL.md) | Repeated findings become automated checks |
| [`data-and-state-safety`](.skills/data-and-state-safety/SKILL.md) | Transactions, concurrency, retention, deletion, and external effects |

### Application correctness

| Skill | Purpose |
|---|---|
| [`frontend-honesty`](.skills/frontend-honesty/SKILL.md) | Never display fabricated, stale, or silently failed state |
| [`verify-through-the-real-path`](.skills/verify-through-the-real-path/SKILL.md) | Prove the real route, artifact, and UI—not only mocked seams |
| [`adversarial-exploit-review`](.skills/adversarial-exploit-review/SKILL.md) | Independent risk-focused review backed by executable evidence |
| [`server-authoritative-state`](.skills/server-authoritative-state/SKILL.md) | Server authority, confirmed mutations, and concurrency preconditions |
| [`clean-migration`](.skills/clean-migration/SKILL.md) | Complete migrations without hidden dual paths |

Each skill is a folder containing `SKILL.md` and optional stack-specific references.
Copy or link only the skills relevant to a project into the agent’s supported skill
directory. Repository rules remain authoritative when they are stricter.

## What success looks like

A healthy AI-assisted codebase is not one where agents produce the most code. It is
one where a new contributor or a fresh agent can answer, quickly and correctly:

- Where does this behavior belong?
- Which contract defines it?
- What evidence proves the change?
- What is the blast radius?
- Which revision is deployed?
- How is temporary state removed?

When those answers are structural instead of tribal, agents remain useful at step
fifty—not only step five.
