# Backend guardrails (Python / SQLAlchemy / Alembic / pytest)

Concrete instances of the hook taxonomy from `docs/appendix/B-hook-taxonomy.md`. Each layer fires earlier than the last; together they make architectural violations structurally impossible or loudly visible.

## The four hook layers

### 1. Pre-write — block before the file is modified
Intercept the agent's write tool call and reject it before bytes hit disk.

- **Cross-module import guard:** reject any import from a sibling module (only the composition root may wire modules together).
- **Shared-kernel guard:** reject ORM models or business logic placed in `shared/` or `common/`.

```python
# arch_guard.py — invoked by a pre-write hook with the target path + new content
import ast, sys

def find_sibling_imports(path: str, src: str) -> list[tuple[int, str]]:
    this_module = path.split("modules/")[1].split("/")[0]  # e.g. "task"
    bad = []
    for node in ast.walk(ast.parse(src)):
        if isinstance(node, ast.ImportFrom) and node.module:
            parts = node.module.split(".")
            if parts[:1] == ["modules"] and len(parts) > 1 and parts[1] != this_module:
                if "# nocheck: arch-guard" not in src.splitlines()[node.lineno - 1]:
                    bad.append((node.lineno, node.module))
    return bad
```

### 2. Pre-commit — block before the commit is created
- **ORM without migration:** an ORM model file is staged with no corresponding Alembic migration. Tests pass (they build the schema via `create_all()`); production crashes (Alembic defines the real schema).
- **Pattern violations:** `len(query.all())` to count rows (loads everything into memory — use `count()`), debug flags left enabled, ORM-specific anti-patterns.

### 3. Pre-push — block before code reaches remote
- **Architecture compliance:** exceptions defined in interface files (they belong in service files); a new service with no corresponding interface.
- **Code quality:** list queries without `order_by` (non-deterministic results); write operations missing foreign-key/ownership validation. A grep for `.scalars().all()` with no `order_by` is a fast first filter; AST-level analysis is what makes it sound.

### 4. Post-edit — run after a file is modified (close the loop)
- **Auto-run tests:** editing `modules/task/service.py` triggers `pytest modules/task/tests/`.
- **Inject failures:** pipe failing test output back into the agent's context so it self-corrects with no human intervention.

## A structured violation message

Every hook prints rule + file + line + fix. The agent receives this as feedback and corrects without re-reading any guide:

```
ARCHITECTURE VIOLATION: cross-module import in modules/task/service.py:12
  from modules.notification.orm import NotificationORM
  → Modules may not import each other's internals. Depend on the interface
    (common/interfaces.py: NotificationServiceInterface) injected at the
    composition root. See docs/04-structural-guidelines.md (Standard 5: Interfaces
    Before Implementation; Standard 7: Guardrails Beat Guidelines).
```

## `# nocheck` with issue traceability

Genuine exceptions are allowed — visibly, intentionally, and temporarily. The suppression must carry an issue reference, and review rejects any that doesn't.

```python
from modules.task.orm import TaskORM  # nocheck: arch-guard — composition root only
from modules.user.orm import UserORM   # nocheck: arch-guard — remove when #42 closes
```

```bash
# Enumerate every live exception in the codebase
grep -rn '# nocheck' modules/ common/

# Pre-push gate: reject suppressions with no issue reference
grep -rn '# nocheck' modules/ \
  | grep -vE '#[0-9]+|composition root' \
  && { echo "Suppression without issue reference — add # <n> or justify."; exit 1; }
```

## The flywheel in practice
Appendix A's rules each began as a single PR comment, became a pattern on the second occurrence, then a rule, then an automated hook: savepoint scope, atomic counter increments, ORM/migration parity, `order_by` on lists, ownership checks on mutations, no open transaction across an external API call, interface/implementation signature parity. See `data-and-state-safety` for the catalog itself; this skill is about converting it into checks.
