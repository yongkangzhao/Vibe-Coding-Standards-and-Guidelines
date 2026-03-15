# Implementation Guide: Staged Rollout

> The standards in this document look like a lot to adopt at once. They aren't meant to be. Agents can't reason about a gradual adoption plan — they need the constraints that exist today to be clear and enforced, and they need new constraints to appear incrementally as the system matures. A staged rollout matches how real teams adopt structure: start with the highest-leverage constraints (module boundaries, interfaces), then layer on automation as patterns emerge from actual PR review.

You don't build all of this at once. A staged approach that works:

**Week 1 — Structure**
- Define module boundaries. No imports between sibling modules.
- Write the interfaces file before building any service.
- Give every module a README with explicit ownership and non-ownership.

**Weeks 1–2 — Tests**
- Require tests before implementation is considered done.
- Use isolated in-memory databases per module.
- Make tests runnable independently per module.

**Weeks 2–3 — Automation**
- Add a pre-write hook that blocks cross-module imports.
- Add a post-edit hook that auto-runs the affected module's test suite.
- Add a pre-commit check for the most common pattern violations.

**Ongoing — Agents and Rules**
- Write agent personas for common tasks (implementer, reviewer).
- Give each persona narrow scope and specific tools.
- Track what keeps appearing in PR reviews. Document as rules. Automate enforcement.

The investment pays off earlier than expected. The first time a guardrail stops an agent from making a mistake you've already fixed once before, it pays for itself.

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

The technology is not the bottleneck. The discipline is.

Module isolation, interface-first design, TDD, bounded contexts — these have been in the software engineering canon for decades. What's new: AI agents don't just benefit from good architecture. They require it.

A human engineer can ask a question when confused, build up context over months, sense when something feels off. An agent starts fresh every session, has no institutional memory, and will confidently implement whatever the local context suggests — even if the global context would say otherwise.

When you build for AI agents, you're not adding something new on top of good engineering. You're taking good engineering seriously enough to make it structural, automatic, and enforced — rather than aspirational, documented, and hoped-for.

The gap between a demo and a production system is exactly that: structure that holds even when nobody is watching, reminding, or reviewing.

---

## The Punchline

If you read this document and thought "this is just good engineering" — you're right.

TDD, module isolation, interface contracts, code review, ownership checks — these have been in the software engineering canon for decades. None of what's described here is new. What's new is the consequence of not doing it.

Human engineers can get away with knowing these practices without enforcing them. They carry context across sessions, build intuition over months, sense when something feels off. They can bend the rules and recover.

AI agents can't. They start fresh every session. They don't sense architectural drift. They don't push back on commands that contradict existing contracts. They will confidently build the wrong thing in the wrong place, at speed, and with no hesitation.

The standards that human teams aspire to but let slide — agents require structurally. The shortcuts that humans recover from — agents compound. The discipline that humans can defer — agents need now.

AI agents don't need a new kind of engineering. They need the engineering we've always known was right, actually enforced — not aspirational, not documented-and-hoped-for, but structural, automatic, and inescapable.

The irony of vibe coding: the thing that makes it work at scale isn't a new invention. It's the old inventions, taken seriously.
