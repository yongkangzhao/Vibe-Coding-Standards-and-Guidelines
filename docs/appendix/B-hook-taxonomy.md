# Appendix B: Hook Taxonomy

Standard 7 says "guardrails beat guidelines." This appendix provides the concrete hook architecture that makes this real. The specific hooks you need depend on your stack — this is one proven taxonomy.

### Pre-write hooks (block before the file is modified)
- **Cross-module import guard**: rejects any import from a sibling module
- **Shared kernel guard**: rejects database models or implementation code in shared/common directories

### Pre-commit hooks (block before the commit is created)
- **ORM without migration**: a database model file is staged without a corresponding schema migration
- **Pattern violations**: `len()` for counting DB rows (loads all into memory), debug flags left enabled, known anti-patterns specific to your ORM

### Pre-push hooks (block before code reaches remote)
- **Architecture compliance**: exceptions defined in interface files (they belong in service files), new service without a corresponding interface
- **Code quality**: list queries without `order_by` (non-deterministic results), write operations without foreign key validation

### Post-edit hooks (run after a file is modified)
- **Auto-run tests**: when a service file is edited, automatically run that module's test suite
- **Inject failures**: pipe test failures back into the agent's context so it self-corrects without human intervention

### CI orchestration guardrails

- **Fail-closed change classifier**: maps a diff to a named verification lane;
  unknown or sensitive paths select the broad lane.
- **Stable aggregate check**: reports selected lane, classifier reason, and final
  outcome through one branch-protection check.
- **Superseded-run cancellation**: stops work whose revision can no longer merge.
- **Exact-tree deduplication**: reuses green evidence only when source tree,
  generated artifacts, toolchain inputs, and relevant configuration are proven identical.
- **Ratcheting budgets**: records current size, complexity, cycles, suppressions,
  duration, and flake baselines; blocks regression while legacy debt is reduced.

The classifier and its tests are sensitive infrastructure. A change to either must
take the broad lane so the mechanism cannot downgrade its own verification.

Each hook outputs a structured message: what rule was violated, which file, which line, what the fix is. The agent receives this as feedback and self-corrects. No human intervention needed for mechanical violations.

### Suppression with traceability

When exceptions are genuinely needed:

```
from sibling_module import Something  # nocheck: arch-guard  # remove when #42 is fixed
```

Every suppression requires an issue reference. Searching for `# nocheck` shows all exceptions in the codebase. Suppressions without issue numbers are rejected in review. This ensures exceptions are intentional, visible, and temporary.
