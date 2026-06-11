---
name: clean-migration
description: When you change an architecture, migrate EVERY call site in the same change and delete the old path — no dual-path codebase, no "backward compat" bypass kwarg, no re-export shim. Use this when making a parameter required, ripping out a concept, promoting duplicated code to one shared copy, renaming a route or type, or choosing between a real migration and a clever in-place patch — for both backend and frontend, even if the user doesn't say "migration."
---

# Clean Migration

> When you change an architecture — a new service pattern, a required argument, a renamed concept, a promoted shared component — migrate every call site in the *same* change and delete the old path. A dual-path codebase is worse than either path alone: the old path silently preserves the exact bug the migration was meant to kill, the new path's guarantees become fictional because the old way is still reachable, and the next reader can't tell which is authoritative. This bites hardest with agents — an agent pattern-matches on whatever it greps first, so a surviving old path is the one it will copy into the next feature.

## When to use this
- Making a parameter required where it was optional, or removing a default that papered over a missing value.
- Ripping out a concept, module, or feature folder — and someone proposes a re-export at the old path "so imports keep working."
- You find the same component/helper/type duplicated across two or three places and want one canonical copy.
- Renaming a route, URL prefix, event name, or type — anything with multiple consumers.
- Choosing between a real schema migration and a clever in-place JSON/string hack.
- A review surfaces "this works but leaves the old way reachable" — an `Optional` bypass, a legacy branch, a shadow type.
- An agent proposes a shim, an adapter, or a `keep_old=True` flag to "reduce risk."
- You're tempted to migrate three of five call sites and leave the awkward two on the old path.

## The standards

**Making something required? Update EVERY call site — no bypass default.** A `param=None` "for backward compat" is not compatibility, it's a permanent hole: the legacy path that triggers when the value is missing is exactly the bug you set out to delete, now reachable forever. Make the argument required and fix every caller in the same change.
```
# ❌ optional bypass — the old, buggy path survives
def assign(user, scope=None):
    if scope is None: return legacy_assign(user)   # bug lives here forever

# ✅ required — every caller migrated in this change
def assign(user, scope): ...
```

**Ripping out a concept? Delete the concept, not just the folder.** A re-export shim at the old path keeps the dead concept alive and importable; agents will keep wiring to it. Move the capability to its real new home and remove the old name entirely — accept that imports must be updated.

**Promoting shared code? One canonical copy, all duplicates deleted, re-introduction structurally forbidden.** Promote to a single shared location, migrate all call sites to it, delete every copy, and add a structural test that asserts the old paths no longer resolve — so a fourth copy can't quietly reappear.
```
test "no duplicate of the shared component":
    matches = find_files("**/<ComponentName>.*")
    expect matches == [ "<shared>/<ComponentName>" ]   # exactly one
```

**Accept the short-term fixture churn.** "Ten tests need updating" is the correct, visible cost of a real migration — not a reason to keep a shim. The alternative isn't less work, it's the bug recurring silently through the surviving path while the tests still pass. Update the fixtures.

**Most-correct over smallest-diff.** Default to the architecturally correct option, not the one that touches the fewest lines. A normalized table beats a JSON-blob hack; a real schema migration beats a clever in-place patch; migrating all call sites beats an optional shim. The smallest diff is usually the one that leaves two paths behind.

## Checklist
- [ ] Every call site of the changed contract is migrated in THIS change — `grep` confirms zero on the old path.
- [ ] No `Optional`/default/`keep_old` kwarg exists solely to preserve the pre-migration path.
- [ ] The removed concept's old name/path is gone — no re-export shim, no adapter, no redirect-forever.
- [ ] Promoted shared code exists in exactly one place; all duplicates are deleted.
- [ ] A structural test forbids re-introducing the old path/duplicate (asserts it doesn't resolve/import).
- [ ] Test fixtures were updated to the new path rather than the migration being narrowed to avoid them.
- [ ] The chosen approach is the most-correct one, not the smallest diff that left two paths alive.

## What breaks without this
- **The bug you migrated to fix recurs silently.** Code still reaches the old path under some input; the new path's guarantee is fictional; tests pass because both paths "work."
- **Agents copy the wrong pattern.** With two patterns present, the next agent greps, finds the legacy one first, and propagates it into new features.
- **Nobody can tell which path is authoritative.** Two ways to do the same thing, no signal which is canonical — every reader re-derives it and some pick wrong.
- **Duplicates drift.** Three copies of a component get fixed one at a time; the others rot, and a styling/security fix lands in one place only.
- **The shim becomes permanent.** "Temporary for backward compat" has no removal trigger; it outlives everyone who remembers why it exists.

## Stack-specific examples
- **Backend:** see `references/backend.md` — required arg over `Optional` bypass, concept removal vs. re-export shim, normalized table vs. JSON hack, a structural test forbidding the old import.
- **Frontend:** see `references/frontend.md` — promoting a triplicated component to one shared copy, migrating all consumers, a vitest structural test that fails on a fourth copy, route/type renames with no redirect shim.

## Related
- `architecture-and-module-boundaries` — the public-surface/barrel boundary is what makes "migrate every call site" tractable; clean migration is how you change that surface without leaving two of them.
- `guardrails-and-rule-flywheel` — the structural test that forbids re-introducing the old path is exactly the kind of guardrail that turns a one-time migration into a permanent invariant.
- `tests-as-ground-truth` — the fixture churn this skill tells you to accept is the same ground-truth suite re-pinning the new contract.
