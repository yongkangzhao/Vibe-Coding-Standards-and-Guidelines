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

The engineers who will get the most out of AI agents aren't the ones with the best prompts. They're the ones who built a codebase where agents can work reliably at step 50, not just step 5.
