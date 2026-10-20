# Backend: Interface-First (Python / Protocol / pytest / SQLAlchemy)

Concrete form of the Spec -> Interface -> Tests -> Implementation loop for a backend module.
Order matters: every file below is written top-to-bottom, and nothing in `service.py` is
written until the README, the Protocol, and the failing tests exist.

## 1. Spec — `modules/organization/README.md`

```markdown
# Organization Module

## Role
Owns organizations, their members, and shared workspace preferences—including the
default locale used for formatting dates, numbers, and generated messages.

## Does NOT own
- User-specific language preferences → `user` module.
- Rendering translated messages → `localization` module.
- User identity / auth → `user` module. We store a member's `user_id`, nothing more.

## Public interface
See `common/interfaces.py :: OrganizationServiceInterface`.
- `get_default_locale(org_id) -> str` — current locale; raises `OrgNotFoundError`.
- `set_default_locale(org_id, locale) -> None` — raises `OrgNotFoundError`, `InvalidLocaleError`.

## Database tables
- `organizations` (orm.py): id, name, default_locale.

## Dependencies
- None on other modules. Pure relationship data.

## Test command
    pytest modules/organization/tests -q
```

## 2. Interface — `common/interfaces.py` (written before `service.py`)

```python
from typing import Protocol
from uuid import UUID


class OrganizationServiceInterface(Protocol):
    # The single, authoritative source of the default locale.
    # If a value or method is not here, it is not part of this service.
    def get_default_locale(self, org_id: UUID) -> str: ...
    def set_default_locale(self, org_id: UUID, locale: str) -> None: ...


class NotificationServiceInterface(Protocol):
    def notify_assignment(self, entity_id: str, user_id: str) -> None: ...
    def notify_completion(self, entity_id: str) -> None: ...
```

Callers in other modules depend on the Protocol, never on the concrete class:

```python
class MessageFormatter:
    def __init__(self, org_service: OrganizationServiceInterface) -> None:
        self._org_service = org_service  # injected at the composition root

    def locale_for(self, org_id: UUID) -> str:
        return self._org_service.get_default_locale(org_id)
```

## 3. Tests — written against the interface, MUST fail first (red phase)

The stub raises `NotImplementedError`, so these tests fail until step 4. Confirm the red
state — a test that passes before implementation is testing nothing. See the
`tests-as-ground-truth` skill and `superpowers:test-driven-development` for the mechanics.

```python
# modules/organization/service.py — stub written so tests import and FAIL
class OrganizationService:
    def get_default_locale(self, org_id: UUID) -> str:
        raise NotImplementedError
    def set_default_locale(self, org_id: UUID, locale: str) -> None:
        raise NotImplementedError
```

```python
# modules/organization/tests/test_service_organization.py
import pytest


def test_get_default_locale_returns_stored_value(org_service, seeded_org):
    assert org_service.get_default_locale(seeded_org.id) == "en-GB"


def test_get_default_locale_raises_for_unknown_org(org_service):
    with pytest.raises(OrgNotFoundError):
        org_service.get_default_locale(uuid4())


def test_set_default_locale_rejects_unknown_locale(org_service, seeded_org):
    with pytest.raises(InvalidLocaleError):
        org_service.set_default_locale(seeded_org.id, "not-a-locale")
```

Run and confirm failure before writing real code:

```bash
pytest modules/organization/tests -q   # expect: failures, all NotImplementedError
```

## 4. Implementation — last

Only now does the agent fill in `service.py` (and the `orm.py` table) until the tests pass.
No new public methods appear that are not already in `OrganizationServiceInterface`; if one is
needed, it goes back to step 1 (update the README and the Protocol first).
