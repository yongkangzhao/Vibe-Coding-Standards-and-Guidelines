# Implementation Guide

> These standards look like a lot to adopt at once. They aren't meant to be. Each stage builds on the last, and each pays for itself before the next one starts. The ordering matters — you can't automate enforcement of rules that don't exist yet, and you can't write meaningful rules until you've seen what actually breaks.

## Staged Adoption

### Stage 1: Boundaries

Before writing any feature code, establish where things go.

- Define module boundaries. Name the modules. Draw the lines.
- Write `interfaces.py` with method signatures — before any `service.py` exists.
- Write a README for each module: what it owns, what it does NOT own.
- No imports between sibling modules. This is the one constraint that matters most early on. If you enforce nothing else, enforce this.

This is the highest-leverage stage. Everything downstream depends on boundaries being clear. Without them, every subsequent stage is built on sand.

### Stage 2: Tests

Once boundaries exist, make them verifiable.

- Require tests before implementation is considered done.
- Tests run against the same database your production uses (not a substitute).
- Tests are runnable per module in isolation.
- Test files follow a predictable naming convention across all modules.

Tests are the agent's feedback loop. Without them, an agent produces code that compiles and returns something plausible — but whether it's correct is unknowable without manually running the full system.

### Stage 3: Automation

Once you've reviewed a few PRs, you know what keeps going wrong. Automate those checks.

- Add a pre-write hook that blocks cross-module imports.
- Add a post-edit hook that auto-runs the affected module's test suite.
- Add a pre-commit check for the most common pattern violations you've seen in review.

Start with the violations you've already caught manually. Each hook is a lesson learned, encoded permanently. Don't try to anticipate every possible violation — automate what you've already seen, and add more as new patterns emerge.

### Stage 4: Team Structure

Once boundaries, tests, and automation are in place, formalize the agent roles.

- Define agent personas with specific responsibilities and tool restrictions.
- Establish the iterative review cycle: implement → review → fix → review → commit.
- Separate test file ownership so agents can work in parallel without conflicts.
- Track what keeps appearing in review. Convert repeated feedback into rules. Automate the rules into hooks.

This is the flywheel. Each rotation makes the next one faster: fewer violations reach review, reviews get shorter, the human's job shifts from catching problems to improving the system.

### The Compounding Effect

Each stage makes the next one cheaper:
- **Boundaries** make tests meaningful (you know what to test in isolation).
- **Tests** make automation possible (you know what "correct" looks like).
- **Automation** makes team structure sustainable (agents self-correct on mechanical violations).
- **Team structure** makes boundaries stronger (adversarial review catches violations that hooks miss).

Skip a stage and the ones above it are fragile. Do them in order and each one is load-bearing.

---

## The Full Loop

Put everything together and the human-AI collaboration looks like this:

```
Human understands architecture
  → gives well-scoped commands
    → agent implements within boundaries
      → human reviews PR
        → catches violations and patterns
          → converts repeated feedback into rules
            → rules get automated in hooks
              → architecture stays healthy
                → human's job stays manageable
```

Break any link and the system degrades. Agents coding without human review produces drift. Humans reviewing without writing rules means the same conversations repeat forever. Humans giving commands without understanding the architecture produces large, disruptive PRs. Automated rules without human evolution become stale.

---

## The Punchline

If you read this document and thought "this is just good engineering" — you're right.

TDD, module isolation, interface contracts, code review, ownership checks — none of this is new. Veteran engineers figured these out decades ago, often the hard way. The practices described here aren't inventions — they're lessons that survived because ignoring them produced real failures in real systems.

What's new is the consequence of not following them.

Human engineers can get away with knowing these practices without enforcing them. They carry context across sessions, build intuition over months, sense when something feels off. They can bend the rules and recover.

AI agents can't. They start fresh every session. They don't sense architectural drift. They don't push back on commands that contradict existing contracts. They will confidently build the wrong thing in the wrong place, at speed, and with no hesitation.

The standards that human teams aspire to but let slide — agents require structurally. The shortcuts that humans recover from — agents compound. The discipline that humans can defer — agents need now.

Veterans who already follow these practices will find AI agents amplify their effectiveness — the structure they built over years becomes the scaffolding that makes agents productive from day one. Teams without that structure will hit the three-month wall faster than any human team ever could, because agents accumulate technical debt at machine speed.

AI agents don't need a new kind of engineering. They need the engineering we've always known was right, actually enforced — not aspirational, not documented-and-hoped-for, but structural, automatic, and inescapable.

The irony of vibe coding: the thing that makes it work at scale isn't a new invention. It's the old inventions, taken seriously.
