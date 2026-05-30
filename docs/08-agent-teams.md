# Standard 14: Agent Teams — Coordinated Multi-Agent Development

> A single agent doing everything — writing code, reviewing architecture, testing edge cases, thinking adversarially — produces inconsistent quality. It optimizes for the last thing it was asked to do and loses focus on the others. Worse, an agent that can both write code and review its own code will approve its own work. Separating roles with enforced tool restrictions (a reviewer that literally cannot write code) produces higher-quality output than any single agent, regardless of how good the prompting is.

> **Standards** (must follow):
> - Separate agent roles with enforced tool restrictions (reviewers cannot write production code)
> - Iterative review cycles — cycle until both test engineer and adversary report clean, not one-shot
> - File ownership to prevent write conflicts between parallel agents
>
> **Guidelines** (recommended):
> - Five-role team structure (lead, planner, test engineer, engineer, adversary)
> - File naming convention for role ownership (`test_service_*` vs `test_security_*`)
> - Context management strategy (kill agents between batches, fresh context per task)
> - "Wait for all reviewers" batching pattern

Standard 9 said "separate roles, separate agents." This standard describes how those agents actually work together at scale — the team structure, the communication protocol, and the operational patterns that emerge when multiple agents collaborate on real codebases.

### The Agent Team

A well-functioning agent team has five roles:

| Role | Can Write Code? | Can Write Tests? | Purpose |
|------|----------------|-----------------|---------|
| **Team Lead** (human or coordinator agent) | No | No | Routes work, batches feedback, decides when to commit |
| **Planner** | No | No | Reads code, produces implementation plans, consulted on-demand |
| **Test Engineer** | No | Yes | Writes tests BEFORE implementation (TDD red phase), reviews coverage after |
| **Engineer** | Yes | No | Makes tests pass, fixes review findings, iterates until clean |
| **Adversary** | No | Yes (exploit tests only) | Thinks like an attacker; writes exploit tests that assert the *secure* outcome — they fail while the flaw exists and pass once it's fixed |

The critical design choice: **test engineer and adversary both write tests, but to different files.** The test engineer owns `test_service_*.py` (correctness tests). The adversary owns `test_security_*.py` (exploit tests). Separate files, no conflicts, safe to run in parallel.

The planner has no write tools. It cannot accidentally modify code while analyzing it. The adversary cannot write production code — only tests that prove flaws. These constraints are enforced at the tool level, not by instructions the agent might ignore.

### The Iterative Review Cycle

The Required Development Loop describes a linear sequence: Spec → Interface → Tests → Implementation. That sequence is the *inner contract* for one module — design before build. The team process here is the *outer review loop* that wraps it: by the time the test engineer writes failing tests below, the spec (README) and interface already exist from the linear loop's first two stages. The outer loop is cyclical:

```
1. [Optional] Planner — consulted at start for complex tasks; confirms the spec (README) and interface are already in place
2. Test Engineer — writes failing tests against the existing interface (TDD red phase)
3. Engineer — implements until all tests pass (TDD green phase)
4. Test Engineer — reviews coverage, adds more tests
5. Adversary — reviews for flaws, writes exploit tests
6. IF issues found → Engineer fixes → back to step 4
7. REPEAT until both reviewers report "all clear"
8. Commit
```

One-shot review misses things. In production systems, the adversary routinely finds issues in round two that the test engineer missed in round one — race conditions, privilege escalation paths, zero-sum violations in financial calculations, bootstrap logic exploits. These are the bugs that tests alone don't catch because they require adversarial thinking about what the code *allows*, not just what it *does*.

An exploit test is written like any correctness test: it asserts the *secure* behavior — a non-owner receives `PermissionError`, a double-spend is rejected — so it fails against the current vulnerable code and passes only once the engineer closes the hole. "Proving a flaw exists" means demonstrating that failure first, then locking in the fix; it does not mean asserting the vulnerability as permanent behavior.

### File Ownership Prevents Conflicts

When two agents write to the same file simultaneously, one overwrites the other's changes. The solution isn't sequential execution (slow) — it's file-level ownership:

```
modules/<name>/tests/
├── conftest.py                    # shared fixtures
├── test_service_<domain>.py       # test engineer's file
└── test_security_<name>.py        # adversary's file (one per module)
```

The naming convention `test_service_*` vs `test_security_*` makes ownership visible at a glance. `grep -r test_security_` finds every exploit test in the codebase. Both agents can run in parallel because they never touch the same file.

For large modules, split `test_service_*.py` further by domain — `test_service_auth.py`, `test_service_crud.py`, `test_service_payments.py`. Target: no test file over 500 lines. A 2,000-line test file is as hard for an agent to navigate as a 2,000-line service file.

### Agent Context Management

Agents accumulate context across tasks. By the fifth task in a session, an agent carries context from tasks one through four — most of it irrelevant. This degrades quality in three measurable ways:

1. **Performance.** Shorter context produces better model output. A focused agent with 2K tokens of relevant context outperforms a bloated agent with 50K tokens of accumulated history. The signal-to-noise ratio drops with every task.
2. **Cost.** Every token in the context window is billed on every subsequent message. An agent carrying 40K tokens of stale context from previous tasks is burning money on irrelevant information with every interaction.
3. **Speed.** Larger contexts take longer to process. Fresh agents with minimal context respond faster.

**Kill agents between batches.** After each commit, shut down all agents and spawn fresh ones for the next batch. Each new agent gets only the context it needs — the relevant files, the failing tests, the issue description. Nothing from previous batches.

The team lead (human or coordinator) retains the high-level context: what's been done, what's next, what decisions were made. The sub-agents stay small and focused. This architecture — persistent lead with ephemeral workers — mirrors how human teams work: the tech lead carries institutional knowledge while individual contributors focus on their current task.

This is counterintuitive — it feels wasteful to throw away context. But the context an agent accumulates is mostly noise. A fresh agent with a precise prompt outperforms a stale agent with a long history — better output, lower cost, faster response.

### The "Wait for All Reviewers" Pattern

When the test engineer finishes first and finds an issue, the temptation is to send the fix to the engineer immediately. Don't. Wait for the adversary to finish too, then batch all findings into a single message.

Why: if you send fixes piecemeal, the engineer does two separate fix rounds instead of one. Each round requires reading context, making changes, running tests. Batching all findings into one message means one round, one context load, one test run.

The exception: if one reviewer is clearly going to take much longer and the other's fix is trivial and independent. But default to batching.

### Reviewers Must Read Code, Not Just Run Tests

A common failure mode: the reviewer runs the test suite, sees all green, and reports "all clear." This is not a review — it's a test run. Tests passing doesn't mean the code is correct. Tests can be wrong. New code can introduce issues that existing tests don't cover.

Every review cycle — including re-reviews after fixes — must include reading the actual changed code. The adversary especially needs to read the implementation to think about new attack vectors. Running tests is verification, not review.
