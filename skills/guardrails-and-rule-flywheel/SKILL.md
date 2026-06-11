---
name: guardrails-and-rule-flywheel
description: Enforce architecture and quality with structural guardrails (hooks, lint rules, CI gates) instead of prose guidelines, keep the main branch behind human-reviewed PRs, and turn every repeated review comment into an automated rule. Use this when setting up or reviewing pre-commit/pre-push/post-edit hooks, husky/lint-staged, ESLint import-boundary or a11y rules, tsc/CI gates, arch-import checks, ORM/migration parity, branch-protection or PR-merge policy, "# nocheck"/eslint-disable suppressions, or whenever the same issue shows up in code review twice and should become a check — on both frontend and backend, even if the user doesn't name it explicitly.
---

# Guardrails & the Rule Flywheel

> A guardrail makes a violation impossible or at least loudly visible; a guideline is a doc an agent may or may not read. Encode constraints in tooling, gate every change behind a human-reviewed PR, and convert each repeated review comment into an automated check. This matters most with AI agents: they start fresh every session with no memory of Monday's PR feedback, don't push back on bad commands, and amplify whatever structure (or chaos) already exists — so the only durable enforcement is structural, not documented.

## When to use this
- Setting up or auditing hooks: pre-write, pre-commit, pre-push, post-edit (backend), or husky + lint-staged (frontend).
- Adding import-boundary enforcement: an arch-guard that rejects cross-module imports, or ESLint `no-restricted-imports` / `import/no-restricted-paths` for feature boundaries.
- Wiring CI gates: running tests, `tsc --noEmit` type-check, lint, and a11y lint (`eslint-plugin-jsx-a11y`) as merge-blocking checks.
- Defining branch-protection / PR policy: no direct pushes to main, every change reviewed before merge.
- Reviewing a suppression (`# nocheck`, `eslint-disable`, `@ts-ignore`) and deciding whether it needs a tracked issue.
- Any moment the same defect appears in review a second time — your cue to write a rule instead of a comment.

## The standards

**Guardrails beat guidelines — enforce in tooling, not prose.** A contributing guide says "don't put business logic in route handlers"; an agent might read it, might decide its case is special. A hook that rejects the write makes the violation structurally impossible. Agents have no institutional memory and no discipline to fall back on, so a rule that depends on "remembering to follow it" is a rule that fails at scale.

**Every guardrail emits a structured, actionable message.** The rejection should name the rule, the file, the line, and the fix — so the agent self-corrects without a human in the loop.
```
ARCHITECTURE VIOLATION: business logic detected in routes/<x>.
  → belongs in modules/<x>/service. See your project's architecture guide.
```

**Main branch is sacred.** No direct pushes; every change lands through a PR with a human review before merge. This is the one structural guarantee that a human sees every change — and agents accumulate changes faster than anyone can casually skim, so without the gate, small violations compound before anyone notices.

**Reviews build the system, not just approve the diff.** Agents code faster than humans read, which creates pressure to rubber-stamp. The answer is never to review less — it's to spend review effort making the next review cheaper by encoding what you found.

**The flywheel: second occurrence → rule → automation.** When the same issue appears in review twice, it's no longer a conversation; it's a candidate for a check that fires before commit, for every agent, forever.
```
mistake → caught in review → fixed → SAME mistake next PR
  → that repetition is the signal → write a rule → automate it → it stops appearing
```

**Suppressions are allowed, but only with traceability.** You can still do the exception — visibly, intentionally, and with a tracked reason. A suppression without an issue reference is rejected in review, so `grep '# nocheck'` (or `eslint-disable`) enumerates every live exception.
```
import Something from "sibling"  // nocheck: arch-guard — remove when #42 closes
```

**Time freed by faster coding goes to planning and review, not more coding.** The bottleneck shifted from writing code to understanding the system well enough to direct agents and verify their output. Running the wrong loop (more code, same review) is how guardrails erode.

## Checklist
- [ ] Architectural rules (import boundaries, layer separation) are enforced by a hook/lint rule, not only documented.
- [ ] Each guardrail prints rule + file + line + fix, not just "rejected".
- [ ] Branch protection forbids direct pushes to main; PRs require human review before merge.
- [ ] CI runs tests, type-check (`tsc --noEmit` / equivalent), lint, and a11y lint as blocking gates.
- [ ] Backend: ORM/migration parity and list-query `order_by` are checked pre-commit/pre-push.
- [ ] Frontend: feature-boundary imports are blocked by ESLint; lint-staged + husky run on staged files.
- [ ] Every suppression (`# nocheck`, `eslint-disable`, `@ts-ignore`) carries an issue reference, enforced in review.
- [ ] Any defect seen in review twice has been converted into a rule (or a tracked issue to add one).

## What breaks without this
- **Drift by amnesia.** A mistake caught on Monday reappears Tuesday from a fresh agent session, because feedback in a PR comment evaporates and nothing fires on the next write.
- **Compounding violations on main.** Without the PR gate, agent-speed changes pile up; by the time a human looks, a small wrong call is woven through a module and needs a large, hard-to-review refactor.
- **Rubber-stamping spiral.** Large PRs are hard to review, so rules get bypassed "just this once" — and once that starts, the rules stop meaning anything.
- **Silent exceptions.** Untracked suppressions accumulate; nobody can answer "where are we cheating, and why" and temporary hacks become permanent.

## Stack-specific examples
- **Backend:** see `references/backend.md` — the four hook layers (pre-write / pre-commit / pre-push / post-edit), an arch-guard violation message, and `# nocheck`-with-issue traceability.
- **Frontend:** see `references/frontend.md` — ESLint `no-restricted-imports` for feature boundaries, husky + lint-staged, `tsc --noEmit` and a11y lint as CI gates, suppression hygiene.

## Related
- Full rationale: `docs/04-structural-guidelines.md` (Standard 7), `docs/06-human-standards.md`, and `docs/appendix/B-hook-taxonomy.md` in this repo.
- `superpowers:requesting-code-review` for running the human/agent review step itself — this skill governs what the review feeds into (rules and gates), not the review mechanics.
- `data-and-state-safety` for the catalog of concurrency, transaction, and ORM mistakes that are exactly the recurring defects worth automating with this flywheel.
