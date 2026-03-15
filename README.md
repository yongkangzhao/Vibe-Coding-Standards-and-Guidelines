# Vibe Coding Standards and Guidelines: An Engineer's Perspective

> AI agents write code fast. These are the engineering standards that determine whether that code holds up.

Vibe coding is a legitimate approach to software development. This is not an argument against it. It's a set of standards for doing it well — drawn from production systems where AI agents and human engineers work together, and from the specific failure modes that appear when they work without structure.

The central premise: AI agents amplify whatever is already true about a codebase. A well-structured codebase with clear boundaries becomes more productive with agents. An unstructured one becomes more chaotic, faster. Everything in this document exists to ensure the former.

---

## Table of Contents

1. [What This Document Is — And What It Is Not](docs/00-what-this-document-is.md)
2. [Standard 1: Start With a Modular Monolith](docs/01-modular-monolith.md)
3. [What Breaks Without These Standards + The Failure Mode in Practice](docs/02-what-breaks.md)
4. [Standard 2: Constraints Define WHERE and HOW](docs/03-constraints.md)
5. [Standards 3–8: The Structural Guidelines](docs/04-structural-guidelines.md)
6. [Standard 9: The Required Development Loop — Spec → Interface → Tests → Implementation](docs/05-development-loop.md)
7. [Standards 10–11: The Human Standards](docs/06-human-standards.md)
8. [Standard 12: Build for Extractability from Day One](docs/07-extractability.md)
9. [Standard 13: Agent Teams — Coordinated Multi-Agent Development](docs/08-agent-teams.md)
10. [Standard 14: Test Infrastructure as Architecture](docs/09-test-infrastructure.md)
11. [Standard 15: Data Retention as an Architectural Decision](docs/10-data-retention.md)
12. [Appendix: Common Mistakes That Become Rules](docs/11-common-mistakes.md)
13. [Implementation Guide: Staged Rollout + The Full Loop](docs/12-implementation-guide.md)

---

The engineers who will get the most out of AI agents aren't the ones with the best prompts. They're the ones who built a codebase where agents can work reliably at step 50, not just step 5.
