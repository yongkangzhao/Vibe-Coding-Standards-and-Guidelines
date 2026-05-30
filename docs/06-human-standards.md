# Standards 11–12: The Human Standards

> Agents don't push back on bad commands. If a human tells an agent to build a feature in the wrong module, the agent builds it there — faithfully, at scale, and at speed. The human role in AI-assisted development is no longer primarily writing code. It's understanding the system well enough to direct agents correctly, and maintaining the structural integrity that agents depend on but cannot maintain themselves.

> **Standards** (must follow):
> - Main branch is sacred — no direct pushes, everything through PRs with human review
> - Every PR requires human review before merge
>
> **Guidelines** (recommended):
> - Specific review process and checklist
> - Rule-writing cadence (convert repeated PR feedback into automated rules)
> - How to invest time savings (planning and review, not more coding)

This is the part that gets skipped in most discussions of AI-assisted development, and it's where things fall apart in practice.

### Main Branch Is Sacred

Non-negotiable: **no direct pushes to main. Everything through a PR. Every PR requires a human review before merge.**

This isn't process for its own sake. It's the one structural guarantee that a human sees every change before it enters the codebase. Agents are fast — changes accumulate quickly. Without a human gate, small violations compound before anyone notices.

Agents code faster than humans can read. This creates real pressure to rubber-stamp PRs or reduce review scope. The answer is not to review less. It's to use reviews to build a system that makes future reviews faster.

### PRs as Rule-Writing Triggers

When the same issue appears in PR review more than once — the same architectural question, the same pattern, the same mistake — that repetition is a signal: **this should be a rule, not a conversation.**

```
Agent makes a mistake
  → Human catches it in PR review
    → Feedback given, PR fixed
      → Same mistake appears in next PR
        → Same feedback given again
          → This is the signal
            → Write a rule
              → Automate the enforcement
                → The mistake stops appearing
```

This is how the rules directory grows. Not by sitting down one day and writing all possible standards from scratch — by treating every repeated PR comment as a candidate for automation. Every time you find yourself saying the same thing in a code review, you're doing work that a hook could do for you permanently.

The result is a system that gets smarter over time. Humans aren't just reviewing — they're teaching. And what they teach gets encoded into automated checks that run on every subsequent PR, for every agent, forever.

### The Commands You Give Matter

Here's the failure mode that doesn't get talked about enough: **if a human gives an agent a command that violates the codebase's architecture, the agent will execute it faithfully, at scale, and at speed.**

Agents don't push back on bad commands. They implement them.

If you don't know the codebase already settled commission rates into the organization module — not because it was the only defensible home, but because that's where the interface put them — and you tell an agent "add commission rate tracking to the platform module," the agent builds it there. Now you have a feature-sized violation, woven through the platform module, contradicting the established contract. Fixing it means a large refactor. Large refactors generate large PRs. Large PRs are hard to review properly. When PRs are hard to review properly, rules get bypassed "just this once." And once that starts, the rules become less meaningful.

The human's job is no longer primarily to write code. It's to understand the system well enough to direct it correctly:

- What each module owns (and explicitly does not own)
- Where the current interfaces are defined
- What architectural decisions have already been made and why
- What scale of change a given command will produce

A human who stays close to the PRs, reads the module READMEs, and understands `interfaces.py` can give precise, well-scoped commands. Those produce small, clean PRs that are easy to review.

A human who hasn't kept up gives vague or contradictory commands. Those produce large, sprawling PRs that touch things they shouldn't, and the review process breaks down.

The bottleneck in a well-run AI-assisted team is not the agent's ability to write code. It's the human's ability to understand the system well enough to direct it.

### Time Savings Go to Planning and Review — Not More Coding

Before agents, coding was the bottleneck. More features meant more engineers writing more code. Agents remove that bottleneck — a single engineer with agents can produce what previously required a team.

The mistake is assuming this means you can ship everything faster with the same amount of planning and review. The bottleneck has shifted.

Coding was never the most important part of engineering. Understanding the problem was. Verifying the solution was. Those two things have always determined whether the code was worth writing at all.

Now that coding is fast, the constraint is the quality of what surrounds it:

- **Planning**: What are the exact module boundaries? What does this interface look like? What are the edge cases? What does this module explicitly NOT own? What does the right answer look like before anyone writes a line?
- **Review**: Is this what was intended? Does it follow the established structure? Are there patterns here that should become rules? Is the architecture still coherent?

A vague command given quickly produces a vague, potentially damaging implementation very quickly. A precise command grounded in architectural understanding produces a precise implementation that fits cleanly. The difference is entirely in the planning.

If you're spending your AI-coding time savings on writing even more code, you're running the wrong loop.
