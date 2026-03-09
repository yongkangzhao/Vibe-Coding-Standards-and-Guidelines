# Vibe Coding Standards and Guidelines: An Engineer's Perspective

> AI agents write code fast. These are the engineering standards that determine whether that code holds up.

Vibe coding is a legitimate approach to software development. This is not an argument against it. It's a set of standards for doing it well — drawn from production systems where AI agents and human engineers work together, and from the specific failure modes that appear when they work without structure.

The central premise: AI agents amplify whatever is already true about a codebase. A well-structured codebase with clear boundaries becomes more productive with agents. An unstructured one becomes more chaotic, faster. Everything in this document exists to ensure the former.

---

---

## What This Document Is — And What It Is Not

Vibe coding has attracted a wave of guides, best practices, and community discussion. The picture that emerges from reading across formal guides, engineering blogs, and developer communities is consistent — and consistently incomplete.

### What the Existing Guides Get Right

The most commonly recommended practices across formal guides are:

- **Prompt engineering**: be specific, layer context, break tasks into small increments, ask before implementing
- **Version control**: use git, commit frequently, revert when the AI breaks something
- **Test after every change**: don't let AI changes accumulate unreviewed
- **Documentation**: maintain READMEs and comments because you didn't write the code line by line
- **Security basics**: don't hardcode credentials, validate inputs, do code review before deployment
- **Simple tech stacks**: choose mature, well-documented technologies that AI tools understand

These are sound practices. None of them are wrong.

### What the Developer Community Is Actually Experiencing

The gap between formal best-practice guides and real developer experiences is significant. Across community forums, the same failure patterns appear independently, described by people who hit them without having read the same guides:

**The three-month wall.** Multiple developers independently describe hitting a point — typically around the three-month mark — where the codebase has grown beyond anyone's ability to understand it. One AI fix breaks ten other things. Red Hat Developer named this "the whack-a-mole effect": *"AI is still just soooooo stupid and it will fix one thing but destroy 10 other things in your code."*

**The unfixable mess.** Developers describe codebases becoming genuinely unrescuable — "functions inside functions, conditions triggering other conditions" — to the point of abandonment. Not because the AI was incompetent, but because there were no structural constraints on where things could go, and the accumulated decisions became incoherent.

**The context window trap.** The widely identified problem is that agents "forget" between sessions. The widely offered solution is to periodically reset sessions or maintain a context file. Neither addresses the root cause: without persistent structure encoded in the codebase itself — interfaces, module boundaries, documented ownership — each new session starts from ignorance.

**The command quality problem.** A less-discussed failure mode: when the human giving commands doesn't understand the architecture well enough to give architecturally correct instructions. The agent executes faithfully. The result is a feature built in the wrong place, which contradicts existing contracts, which requires a large refactor, which generates a PR too large to review properly. In a survey of 18 CTOs, 16 reported production disasters directly caused by AI-generated code. The AI wrote what it was asked. The commands were the problem.

### The Spectrum of Community Positions

The community has split into two camps, both of which miss something important:

**The anti-vibe-coding camp** (dominant in communities like r/programming and r/cscareerquestions) holds that vibe coding is fundamentally incompatible with professional engineering — a "red flag in interviews," evidence of undisciplined thinking, and incompatible with team collaboration. This position is wrong not because the criticisms are false, but because it treats the failure modes as inherent to AI-assisted development rather than as consequences of absent structure.

**The pro-vibe-coding camp** (dominant in builder and founder communities) holds that vibe coding is a superpower that democratizes software creation and makes traditional engineering concerns obsolete. This position is wrong because it has not yet experienced the three-month wall, or has not yet tried to hand the project to someone else, or has not yet needed to change something foundational.

The most accurate framing comes from Addy Osmani's distinction between "vibe coding" and "AI-assisted engineering": vibe coding is for throwaway experiments; AI-assisted engineering integrates AI into a structured development process with human oversight of architecture and review. This document extends that framing by providing the specific structural standards that make AI-assisted engineering concrete and actionable.

### What They All Miss

Every guide and community discussion operates at the level of workflow habits and session management. They describe how to interact with an AI tool more effectively within whatever structure you already have. None of them describe what structure to build in the first place.

Nobody names the architecture and explains why it's the right choice. Nobody defines interface contracts as the mechanism that closes architectural debates permanently. Nobody describes module isolation as the structural prevention of logic leakage and feature duplication. Nobody explains how PR reviews should feed back into automated enforcement rules. Nobody describes the development loop as a required sequence — spec first, interface second, tests third, implementation last — that determines the quality of everything after it.

**The result: guidance that helps you prompt better today, but leaves you unprepared for month three.**

A consistent pattern across all existing guidance: vibe coding is framed as appropriate for prototypes and MVPs, implying that production systems require returning to traditional methods. **This document rejects that framing.** Vibe coding is viable for production — when the right architectural standards are in place. The problem is not the AI. The problem is the absence of structure.

### What This Document Is

A set of numbered engineering standards covering:
1. Which architecture to start with and why
2. The exact module structure and the rationale for uniformity
3. The required development loop: Spec → Interface → Tests → Implementation
4. The difference between guidelines (advisory) and guardrails (structural enforcement)
5. The human role: planning, commanding correctly, and converting PR patterns into automated rules
6. How today's decisions enable microservice extraction later

### What This Document Is Not

- **A prompt engineering guide.** Prompting technique is a separate topic. This document addresses what the codebase looks like, not how you describe what you want.
- **Tool-specific.** These standards apply to any AI coding assistant — Cursor, Claude Code, Copilot, or others.
- **Anti-vibe-coding.** The goal is not to write everything by hand. The goal is to build a codebase where agents can work reliably at step 50, not just step 5.
- **A guide for throwaway demos.** If you are building a prototype that will be discarded in a week, most of this is overkill. This is for projects that need to grow.
- **Complete.** These standards reflect patterns proven effective in production systems. They are a starting point, not a ceiling.

---

## Standard 1: Start With a Modular Monolith

There are many ways to structure a software project. Microservices, serverless, classic monolith, MVC, event-driven — each has legitimate use cases. But for a vibe-coded project specifically, the choice matters more than usual because AI agents interact with codebases differently than humans do: no accumulated context, bounded sessions, no institutional memory. They need explicit structure.

Here's how the common options compare:

| Architecture | Agent Visibility | Domain Isolation | Early Complexity | Testable in Isolation | Path to Scale |
|---|---|---|---|---|---|
| **Classic Monolith** | Full | None — everything leaks | Low | Hard | Rewrite required |
| **Modular Monolith** ✓ | Full | Enforced — module boundaries | Low-Medium | Yes — per module | Extract modules to services |
| **Microservices** | Partial — split across services/repos | Strong — network boundary | Very High | Yes — per service | Already there |
| **Serverless** | Fragmented — logic scattered across functions | Minimal | Medium | Hard — mocking required | Vendor-dependent |
| **MVC / Layered** | Full | Weak — by technical type, not domain | Low | Hard — layers coupled | Rewrite required |

**Classic monolith**: no ceremony, fast to start, but no isolation. Agents put things wherever seems locally reasonable. Two agents, two different answers about where something belongs. Logic leaks everywhere.

**Microservices**: strong domain isolation, but far too much overhead for early stage. Agents have to reason about network failures, distributed transactions, service discovery, and multiple repos at once. Worse, the domain boundaries you define in week one are usually wrong — and moving a microservice boundary is expensive. Moving a module boundary is just moving files.

**Serverless**: fine for specific use cases (background jobs, webhooks). As a primary architecture, too granular. No module concept to anchor around. Agents lose the big picture across hundreds of functions.

**MVC/Layered**: organizes by technical type (models, controllers, services) not by domain. A user management feature touches the models layer, the controllers layer, and the services layer — all of which are shared with every other feature. No isolation, wide blast radius for any agent change.

**Modular monolith**: single codebase (full agent visibility), enforced domain boundaries (structural, not conventional), low early complexity, per-module testability, and a clear path to microservices when scale actually demands it. The modular monolith and microservices architecture share the same underlying decisions — interface contracts, dependency injection, domain ownership. The monolith is just the right first deployment target.

Start here. The rest of this article describes how to build one that holds up.

---

## What Breaks Without These Standards

Before getting to solutions, let's name the failure modes precisely, because they matter:

**Feature duplication.** The same logic gets implemented in multiple places. A function that fetches a user's active subscription exists in three files, each written at a different time, each slightly different. Billing logic changes. One of them gets updated.

**Logic leakage.** Business rules that belong in one layer appear in all of them. The discount calculation lives in the API handler, in the frontend component, and hardcoded into a database query. Each has subtly different rounding behavior. None agree.

**Inconsistent patterns.** Some endpoints paginate using `len()` (loads the full table into memory). Others use `COUNT(*)`. Some fetch related data in a loop, one query per item. Others batch. The codebase doesn't have a standard — it has a history.

**Invisible regressions.** A change silently breaks something else. There's no fast feedback loop. You find it in production.

**Architectural drift.** The intended structure lives in a document or in someone's head. The actual structure is what the code does. These diverge over time — and faster when agents are involved, because agents start fresh every session. No institutional memory.

None of this is new. Every team that has grown a codebase without discipline has hit this. What's new is the speed. Agents are productive and confident even when they're wrong, which means a codebase can get large and inconsistent very quickly.

---

## The Failure Mode in Practice

Imagine you're building a marketplace platform — businesses transact, the platform earns a commission. Different businesses have different rates. Simple enough.

Now the question: **where does the commission rate live?**

Three defensible options:

| Option | Location | Rationale |
|---|---|---|
| A | Platform module | The platform sets and owns rates; it's a revenue concern |
| B | Organization module | Rates are part of the business relationship; orgs have rates |
| C | New billing/contracts module | This is its own domain; neither platform nor org should own it |

A senior engineer could write a convincing case for any of them.

Here's what happens without a decision mechanism:

One developer starts building the commission feature and puts it in the platform module. Another, working on a different feature, needs a commission rate and looks in the organization module — makes sense, it's an org-level setting. It's not there, so they add it. A third agent, writing a billing report, finds rate data in both places with different values and doesn't know which to trust. It picks one. Now you have three implementations, two of which are wrong, and every PR that touches commission rates becomes an architectural argument.

Here's what happens with an interface contract:

```python
# common/interfaces.py — written before any implementation

class OrganizationServiceInterface(Protocol):
    def get_commission_rate(self, org_id: str) -> Decimal: ...
```

Three lines. Now:

- Every agent that needs a commission rate calls `org_service.get_commission_rate(org_id)` — because that's the only place it exists
- Every PR is checked against the interface — code that doesn't use it is wrong by definition
- The architectural debate is closed — not because everyone agreed it was the best choice, but because the interface made the choice structural

The critical insight here: **the interface's value isn't that it found the right answer. It's that it ended the question.** The cost of endless indecision — constant redesign, conflicting implementations, agents guessing — is higher than the cost of a suboptimal-but-consistent decision.

Architecture is full of questions that don't have one correct answer. The dangerous ones aren't the hard questions. They're the reasonable questions that keep getting re-asked, re-litigated, and re-implemented by each person (or agent) who encounters them fresh.

---

## Standard 2: Constraints Define WHERE and HOW — Not WHAT

This is the thing that gets misunderstood about architectural constraints.

People hear "you can't import from a sibling module" and think: restriction. Limitation. Something in the way.

That's not what it is.

A constraint that says "business logic goes in `service.py`, database tables go in `orm.py`, public contracts go in `interfaces.py`" doesn't limit what you can build. It answers a question that would otherwise be asked again and again by every person — and every agent — that touches the codebase.

**Where does this code go?** The constraint answers it. Always the same answer. The question stops being asked.

For human engineers, this saves time and reduces friction. For AI agents, it's more fundamental: it collapses the search space. An agent that doesn't know where something belongs has to explore the entire codebase, make inferences, and guess. An agent that knows "service files contain business logic, and service files are always at `modules/<name>/service.py`" goes directly to the right place. Every time. For every module.

---

## Standards 3–8: The Structural Guidelines

These are drawn from production systems. Examples use a generic task management app — `task`, `user`, `notification` modules. Simple enough to be obvious, concrete enough to be useful.

### 1. Predictable Structure Over Clever Structure

Every module looks exactly the same:

```
modules/task/
├── __init__.py       # public exports only
├── service.py        # all business logic
├── orm.py            # database tables
├── README.md         # design document
└── tests/
    └── test_task_service.py
```

This is deliberately boring. Clever code has cognitive overhead — unusual patterns require explanation. Boring, predictable code has zero overhead. An agent (or a new team member) that understands one module understands all of them. Business logic is always in `service.py`. Tables are always in `orm.py`. Public API is always in `__init__.py`.

When agents don't know where to look, they look everywhere and make assumptions. Predictable structure is the navigation system.

### 2. Module READMEs as Executable Specs

For humans, documentation helps. For agents, documentation is the specification.

Every module README should cover:

- **Role**: one paragraph, what this module owns
- **What it does NOT own**: explicit non-ownership (just as important)
- **Public interface**: every method, what it does, what it raises
- **Database tables**: what data it stores
- **Dependencies**: what other services it requires
- **Test command**: how to run this module's tests in isolation

The "does NOT own" section is particularly important. Without it, agents optimize locally — they put things in the nearest module that seems plausible. With it, agents get a clear answer when they ask "should this go here?" — and sometimes the answer is explicitly "no, that belongs to `notification`."

A module without a README is incomplete, regardless of whether the code works.

### 3. Interfaces Before Implementation

Before any implementation is written, the contract should exist:

```python
# common/interfaces.py — written first, before service.py exists

class TaskServiceInterface(Protocol):
    def create_task(self, data: TaskCreate) -> TaskResponse: ...
    def assign_task(self, task_id: UUID, user_id: str) -> TaskResponse: ...
    def complete_task(self, task_id: UUID) -> TaskResponse: ...
    def list_tasks(self, user_id: str, limit: int, offset: int) -> Tuple[List[TaskResponse], int]: ...

class NotificationServiceInterface(Protocol):
    def notify_assignment(self, entity_id: str, user_id: str) -> None: ...
    def notify_completion(self, entity_id: str) -> None: ...
```

This does several things at once:

- **Defines scope.** If a method isn't in the interface, it isn't part of this service. No ambiguity.
- **Enables parallel work.** Agent A implements the task module. Agent B implements notification. They never coordinate directly — both work against the same interfaces file. When they're done, the composition root wires them together and they fit.
- **Makes capabilities legible.** One file. Read it. Know what the entire system can do.

The interface is the AI-to-AI communication protocol.

### 4. Tests as Ground Truth

For humans, tests catch regressions. For agents, tests define correct behavior in machine-readable form.

```python
def test_assign_task_raises_if_user_not_found(task_service):
    task = task_service.create_task(TaskCreate(title="Fix the bug"))
    with pytest.raises(UserNotFoundError):
        task_service.assign_task(task.task_id, "nonexistent-user-id")

def test_complete_task_triggers_notification(task_service, mock_notification_service):
    task = task_service.create_task(TaskCreate(title="Write the tests"))
    task_service.assign_task(task.task_id, "user-123")
    task_service.complete_task(task.task_id)
    mock_notification_service.notify_completion.assert_called_once_with(str(task.task_id))
```

Without tests, an agent has no feedback loop. It produces code that compiles and returns something plausible — but whether the behavior is correct is unknowable without running the full system manually.

With tests — especially written before implementation — agents work against a concrete, unambiguous spec. Failures are immediate, specific, and self-correcting. A post-edit hook that automatically runs the module's test suite and injects failures back into the agent's context closes the loop entirely. The agent writes code, tests run, failures appear, agent corrects. No human required for basic correctness verification.

Tests are not quality assurance for agents. They are the ground truth.

### 5. Guardrails Beat Guidelines

Here's the distinction that matters most in practice.

A contributing guide that says "don't put business logic in route handlers" is a **guideline**. An agent might read it, might not, might decide this case is special.

A hook that intercepts every file write and rejects it if business logic appears in a route handler is a **guardrail**. The agent cannot violate the rule — the tool call is rejected. It receives:

```
ARCHITECTURE VIOLATION: Business logic detected in routes/task.py.
Business logic belongs in modules/task/service.py.
See docs/architecture.md for module structure.
```

The agent knows what it did wrong, why, and exactly what to do instead. No re-reading the contributing guide. No judgment call about whether this case is an exception.

The distinction matters at scale. Guidelines work when everyone reads them, remembers them, and consistently chooses to follow them. Guardrails work regardless. They don't rely on memory or discipline — they make violations structurally impossible or at minimum explicit and visible.

When exceptions are genuinely needed:

```python
from modules.task.orm import TaskORM  # nocheck: arch-guard
# This is the composition root — the only valid cross-module import location
```

You can still do the thing. But you do it intentionally, visibly, and with an explanation. `grep '# nocheck'` shows every exception in the codebase.

**Encode constraints in tooling, not just documentation.**

### 6. Bounded Scope Creates Reliable Agent Work

The best agent tasks have a defined start (read these files), a clear scope (touch only these files), and a verifiable end (tests pass, review checks clear). "Implement the notification module" is a great agent task. "Improve the app" is not.

Architecture designed so most feature work lives within a single module creates natural, right-sized work units. When that's true, agent tasks are bounded, predictable, and auditable. When it isn't — when adding a feature requires changes scattered across five modules — agent tasks become unpredictable, the judgment calls interact, and the result is hard to review and hard to roll back.

### 7. Separate Roles, Separate Agents

The same agent should not write code, review architecture, and do code quality review. A well-defined agent system has distinct roles:

- **Implementer**: follows the TDD workflow, builds one module at a time, touches nothing else
- **Architecture reviewer**: reads code, produces a compliance report, cannot write code
- **Code quality reviewer**: finds bugs, performance issues, security problems, produces prioritized findings

These roles are enforced — the architecture reviewer literally has no write tools available. It cannot accidentally fix things while reviewing them. An agent with a narrowly scoped role does that role reliably.

Give agents one job.

---

## Standard 9: The Required Development Loop — Spec → Interface → Tests → Implementation

The seven principles describe the structure. This is the process — the order of operations for every new module or feature:

```
1. Spec        — README: what does this module own? what does it NOT own?
2. Interface   — method signatures in interfaces.py, before any implementation
3. Tests       — write against the interface; confirm they FAIL (red phase)
4. Implementation — make tests pass, within the constraints already set
```

By the time you write implementation code, three layers of constraints already exist. Implementation's only job is to satisfy them.

**Spec first.** The README isn't documentation written after the fact — it's the design decision that happens before any code. It answers "what belongs here?" and crucially, "what does NOT belong here?" This is where the commission rate debate gets resolved. Once it's in the spec, the interface encodes it, and the question is closed.

**Interface second.** Not the implementation — just the shape:

```python
# common/interfaces.py — written before service.py exists

class NotificationServiceInterface(Protocol):
    def notify_assignment(self, entity_id: str, user_id: str) -> None: ...
    def notify_completion(self, entity_id: str) -> None: ...
```

This locks scope (if it's not in the interface, it's not part of this module), enables parallel agent work (other modules build against this before the implementation exists), and gives the implementer agent a concrete target.

**Tests third — and they must fail.** Write tests against the interface before implementation exists. Every test should fail (stubs raise `NotImplementedError`). If tests pass before implementation, the tests are wrong. Confirm the red state — it's not a formality, it's proof that the tests describe real behavior.

**Implementation last.** The agent reads the interface, reads the failing tests, and writes code until they pass. There is no design work left. There is only build work. This is the constraint that keeps agents focused and prevents them from inventing architecture mid-implementation.

The critical point for AI agents specifically: a human engineer might skip the spec and rely on domain knowledge. An agent starts fresh every session — it has no internalized context. Without spec, interface, and tests already in place, the agent designs on the fly, and its design reflects local context, not the global architecture decisions that have already been made.

**Design is a human activity. Implementation is what agents do.**

---

## Standards 10–11: The Human Standards

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

If you don't know that commission rates live in the organization module and you tell an agent "add commission rate tracking to the platform module," the agent builds it there. Now you have a feature-sized violation, woven through the platform module, contradicting the established contract. Fixing it means a large refactor. Large refactors generate large PRs. Large PRs are hard to review properly. When PRs are hard to review properly, rules get bypassed "just this once." And once that starts, the rules become less meaningful.

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

---

## Standard 12: Build for Extractability from Day One

The biggest objection to careful early architecture is "we're too small for this." In a narrow sense, that's true. A small team doesn't need microservices.

But the decisions you make in a modular monolith today are the same decisions you'd make in a microservices architecture. They're not premature — they're early.

Return to the commission rate example. If the platform eventually grows large enough that transaction processing needs its own service:

```python
# Today — local module call
self.org_service: OrganizationServiceInterface = OrganizationService(db)

# Tomorrow — remote service call
self.org_service: OrganizationServiceInterface = OrganizationHTTPClient(
    base_url=settings.ORG_SERVICE_URL
)
```

One line changes in the composition root. Nothing else changes. Every module that calls `self.org_service.get_commission_rate(org_id)` keeps working identically, because they were always calling an interface, not a concrete class.

This works because:
1. Every service implements a `Protocol` interface, not a class
2. Cross-module dependencies are injected at a single location (the composition root)
3. Modules use opaque string identifiers, not database foreign keys — no ORM coupling to unpick

A codebase built this way is microservice-ready from day one. Not because microservices were the target, but because module isolation and interface contracts are the same idea at different scales.

---

## Implementation Guide: Staged Rollout

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

The engineers who will get the most out of AI agents aren't the ones with the best prompts. They're the ones who built a codebase where agents can work reliably at step 50, not just step 5.
