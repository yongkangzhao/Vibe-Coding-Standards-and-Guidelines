# Vibe Coding Standards and Guidelines: An Engineer's Perspective

> AI agents write code fast. These are the engineering standards that determine whether that code holds up.

Vibe coding — building software by directing AI agents in natural language rather than writing most of the code by hand — is a legitimate approach to software development. This is not an argument against it. It's a set of standards for doing it well — drawn from production systems where AI agents and human engineers work together, and from the specific failure modes that appear when they work without structure.

The central premise: AI agents amplify whatever is already true about a codebase. A well-structured codebase with clear boundaries becomes more productive with agents. An unstructured one becomes more chaotic, faster. Everything in this document exists to ensure the former.

---

## Table of Contents

1. [What This Document Is — And What It Is Not](docs/00-what-this-document-is.md)
2. [What Breaks Without These Standards](docs/02-what-breaks.md)

**Architecture**

3. [Standard 1: Start With a Modular Monolith](docs/01-modular-monolith.md)
4. [Standard 2: Constraints Define WHERE and HOW](docs/03-constraints.md)
5. [Standards 3–9: The Structural Guidelines](docs/04-structural-guidelines.md)

**Process**

6. [Standard 10: The Required Development Loop](docs/05-development-loop.md)
7. [Standards 11–12: The Human Standards](docs/06-human-standards.md)

**Scaling**

8. [Standard 13: Build for Extractability from Day One](docs/07-extractability.md)
9. [Standard 14: Agent Teams — Coordinated Multi-Agent Development](docs/08-agent-teams.md)
10. [Standard 15: Test Infrastructure as Architecture](docs/09-test-infrastructure.md)
11. [Standard 16: Data Retention as an Architectural Decision](docs/10-data-retention.md)

**Putting It Together**

12. [Implementation Guide](docs/11-implementation-guide.md)

**Appendix**

- [A: Common Mistakes That Become Rules](docs/appendix/A-common-mistakes.md)
- [B: Hook Taxonomy](docs/appendix/B-hook-taxonomy.md)

---

## Use these standards as skills

This repo also ships the standards as a set of six **Claude Code skills** that trigger automatically during development — stack-agnostic, for frontend and backend alike. The prose above is the rationale; the skills are the operational version an agent applies while it works.

| Skill | What it enforces | Standards |
|---|---|---|
| [`architecture-and-module-boundaries`](skills/architecture-and-module-boundaries/SKILL.md) | Modular-monolith structure, domain modules, enforced boundaries, day-one extractability | 1–3, 13 |
| [`interface-first-development`](skills/interface-first-development/SKILL.md) | READMEs as specs + contracts before code; the Spec → Interface → Tests → Implementation loop | 4, 5, 10 |
| [`tests-as-ground-truth`](skills/tests-as-ground-truth/SKILL.md) | Tests as the executable spec, run against real (not substitute) infrastructure | 6, 15 |
| [`agent-team-review-loop`](skills/agent-team-review-loop/SKILL.md) | Bounded tasks, tool-restricted roles, iterate-until-clean review | 8, 9, 14 |
| [`guardrails-and-rule-flywheel`](skills/guardrails-and-rule-flywheel/SKILL.md) | Structural enforcement over docs; PR gate; repeated feedback → automated rules | 7, 11, 12 |
| [`data-and-state-safety`](skills/data-and-state-safety/SKILL.md) | Soft-delete + legal erasure, and the recurring persistence pitfalls | 16, App. A |

Each skill is a self-contained folder under [`skills/`](skills/) — a `SKILL.md` plus a `references/` directory. The body stays language-neutral; concrete examples live in `references/backend.md` (Python / SQLAlchemy / pytest) and `references/frontend.md` (TypeScript / React / vitest).

**Use them:** copy (or symlink) any `skills/<name>/` folder into a directory Claude Code reads:

- `~/.claude/skills/` — available in every project you work on, or
- a project's `.claude/skills/` — that project only.

Claude Code auto-discovers each `SKILL.md` from there — no plugin or install step.

### Generic webapp skills (`.skills/`)

The six skills above carry this document's *architecture* (modular monolith, interface-first, agent teams). The [`.skills/`](.skills/) folder ships a second, **architecture-agnostic** set: the correctness-and-trust disciplines that apply to *any* web application — server or client, monolith or not. They were distilled from the failure modes that actually reach production when agents move fast: a UI that renders a plausible lie, a green test suite over a broken route, an optimistic update that hides a failure.

| Skill | What it enforces |
|---|---|
| [`frontend-honesty`](.skills/frontend-honesty/SKILL.md) | The UI never displays something untrue — absent ≠ `$0.00`, error ≠ empty, no raw identifiers or echoed backend errors, formatters that reject garbage before coercion, no fabricated stats |
| [`verify-through-the-real-path`](.skills/verify-through-the-real-path/SKILL.md) | "It works" must be proven through the real request path + the deployed artifact + the genuine UI — not inferred from unit tests that mocked the exact seam that breaks |
| [`adversarial-exploit-review`](.skills/adversarial-exploit-review/SKILL.md) | Findings are failing exploit tests, not prose; an independent reviewer; ≥2 reviewers with distinct failure lenses on money/auth/migration/data-loss; mutation-test the guards |
| [`server-authoritative-state`](.skills/server-authoritative-state/SKILL.md) | The client is a cache of server truth — no optimistic flip that masks a failure, cache written from confirmed responses, re-entrancy guards, no client-derived authority |
| [`clean-migration`](.skills/clean-migration/SKILL.md) | Change an architecture → migrate every call site in the same change and delete the old path; no dual-path shim, no "backward compat" bypass kwarg |

Same layout (`SKILL.md` + `references/{backend,frontend}.md`) and the same install: copy any `.skills/<name>/` into `~/.claude/skills/` or a project's `.claude/skills/`.

---

The engineers who will get the most out of AI agents aren't the ones with the best prompts. They're the ones who built a codebase where agents can work reliably at step 50, not just step 5.
