# Implementation Guide: Staged Rollout

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
