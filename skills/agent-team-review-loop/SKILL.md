---
name: agent-team-review-loop
description: Structures multi-agent work into bounded tasks, tool-restricted roles, and an iterative review cycle that loops until reviewers report clean. Use this when dispatching agents to build or change code (frontend or backend), when an agent is about to both write and review its own work, when scoping a task like "implement the notification module" vs "improve the app", when two agents might write the same test file, when an agent builds a React feature and then needs an independent pass for accessibility or business-logic leak, or when deciding whether to commit after one review pass — even if the user doesn't name it explicitly.
---

# Agent-Team Review Loop

> Give every agent one bounded job, restrict its tools so a reviewer literally cannot approve its own code, and cycle the review (test-engineer then adversary) until both report clean — not one-shot. This matters most when agents write the code: an agent that can both write and review will rubber-stamp itself, a vague task produces vague output, and an agent five tasks deep is mostly carrying noise. Bounded scope, separated roles, and fresh context are what make multi-agent work auditable instead of chaotic.

## When to use this
- Dispatching one or more agents to build or modify a module — backend (service, ORM, transactions) or frontend (components, hooks, presentation).
- Scoping an agent task: deciding whether "implement X module" is bounded enough, or whether it's secretly "improve the app".
- An agent is about to write code AND review the same code — split it.
- A frontend feature needs an independent reviewer for accessibility and to confirm no business logic (validation, pricing, authz) leaked into the presentation layer.
- Two agents will write tests for the same module and you need them to not clobber each other.
- Deciding whether to commit after a single review pass (don't — loop until clean).
- Setting up parallel agents and needing a file-ownership convention so they don't collide.

## The standards

**Bound every agent task: defined start, narrow scope, verifiable end.** A good task says "read these files, touch only these files, done when these tests pass and review clears." Agents start fresh with no institutional memory, so an open-ended task ("improve the app") produces unpredictable, scattered, unreviewable diffs. Architect work so most features live in one module — then each task is naturally right-sized.
```
GOOD: "Implement notification module. Read README+interface. Touch modules/notification/*. Done when test_service_notification passes + reviews clear."
BAD:  "Improve notifications."   # no start, no scope, no end
```

**Separate roles into separate agents.** One agent that writes, reviews architecture, and hunts bugs optimizes for whatever it was asked last and loses the others. Distinct roles — planner, test-engineer, engineer, adversary — each do their one job reliably. This mirrors the 7-step team workflow in `~/.claude/CLAUDE.md`.

**Enforce role boundaries at the tool level, not by instruction.** A reviewer with no write tools cannot fix-then-approve its own work; an adversary that can only write test files cannot "helpfully" patch production code. Instructions get ignored; absent tools cannot be. The planner and reviewers are read-only or test-only by construction.
```
planner:       read-only
test-engineer: read + write tests (test_service_*)
engineer:      read + write production code + tests
adversary:     read + write exploit tests only (test_security_*)
```

**Review iteratively, not once.** Engineer makes tests pass; then test-engineer reviews coverage; then adversary reviews for flaws; if either finds issues, engineer fixes and the cycle repeats — commit only when BOTH report clean. One-shot review misses the round-two bugs (race conditions, privilege escalation, financial zero-sum violations) that need adversarial thinking, not just a green test run.
```
loop:
  engineer implements -> tests green
  test-engineer reviews coverage  (reads code, adds tests)
  adversary reviews for flaws      (reads code, writes exploit tests)
  if findings -> engineer fixes -> repeat
until both clear -> commit
```

**Own files by name so parallel agents never collide.** Test-engineer writes `test_service_*`, adversary writes `test_security_*` — separate files, safe to run in parallel, ownership visible at a glance. When two agents write the same file, one silently overwrites the other.

**Batch all reviewer findings into one fix round.** Wait for both reviewers to finish, then hand the engineer a single combined list. Piecemeal fixes mean multiple context loads and test runs for what could be one pass.

**Review means reading the changed code, not just running tests.** Green tests prove the tested paths work, not that the code is correct or safe — the adversary especially must read the implementation to find new attack surface. Every cycle, including re-reviews after fixes, reads the diff.

**Kill agents between batches; the lead keeps the high-level context.** After each commit, shut down workers and spawn fresh ones with only what the next task needs. Accumulated context is mostly noise — it degrades output quality, costs tokens on every message, and slows responses. The lead (human or coordinator) retains "what's done / what's next"; workers stay small and ephemeral.

## Checklist
- [ ] Each agent task has a defined start (files to read), narrow scope (files it may touch), and verifiable end (named tests + review clear).
- [ ] Reviewers (planner, adversary) have NO production-write tools; adversary writes test files only.
- [ ] Test-engineer's tests and adversary's exploit tests live in separate files (`test_service_*` vs `test_security_*`).
- [ ] Review ran at least twice and both test-engineer AND adversary reported clean before commit.
- [ ] Every review (including re-reviews) read the changed code, not just the test output.
- [ ] Findings from both reviewers were batched into one fix round.
- [ ] Worker agents were killed/replaced between commits; only the lead carried context forward.

## What breaks without this
- An agent that writes and reviews its own code approves itself — flaws ship because nothing independent ever objected.
- One-shot review passes the test-engineer's coverage check but misses the race condition or privilege-escalation path the adversary would have caught in round two.
- Two parallel agents write the same test file; one overwrites the other's tests and coverage silently regresses.
- A vague "improve the app" task produces a sprawling diff across five modules — unreviewable, hard to roll back, full of interacting judgment calls.
- A worker agent five tasks deep carries 40K tokens of stale context: worse output, higher cost, slower responses.

## Stack-specific examples
The process is stack-agnostic. Backend reviewers focus on security, transactions, and authorization; frontend reviewers focus on accessibility and ensuring no business logic (validation, pricing, authz) leaks into the presentation layer — that belongs in the backend.
- **Roles, tool-permission table, file-ownership convention, and the batching / kill-between-batches patterns:** see `references/roles.md`

## Related
- Full rationale: `docs/04-structural-guidelines.md` (Standard 8: Bounded Scope, Standard 9: Separate Roles) and `docs/08-agent-teams.md` (Standard 14) in this repo.
- `superpowers:subagent-driven-development` and `superpowers:dispatching-parallel-agents` for the mechanics of running independent agents — reference them; don't duplicate.
- `superpowers:requesting-code-review` and `superpowers:receiving-code-review` for review mechanics and how to respond to findings with rigor.
- Aligns with the 7-step team workflow in `~/.claude/CLAUDE.md` (Planner, Test-engineer, Engineer, Adversary).
