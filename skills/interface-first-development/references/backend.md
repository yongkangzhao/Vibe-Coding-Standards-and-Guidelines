# Backend: Interface-First (Python / Protocol / pytest / SQLAlchemy)

Concrete form of the Spec -> Interface -> Tests -> Implementation loop for a backend module.
Order matters: every file below is written top-to-bottom, and nothing in `service.py` is
written until the README, the Protocol, and the failing tests exist.

## 1. Spec — `modules/organization/README.md`

```markdown
# Organization Module

## Role
Owns the business relationship: organizations, their members, and their
contractual settings — including the commission rate the platform charges.

## Does NOT own
- Payment processing or invoicing → `billing` module.
- Notifications on rate changes → `notification` module.
- User identity / auth → `user` module. We store a member's `user_id`, nothing more.

## Public interface
See `common/interfaces.py :: OrganizationServiceInterface`.
- `get_commission_rate(org_id) -> Decimal` — current rate; raises `OrgNotFoundError`.
- `set_commission_rate(org_id, rate) -> None` — raises `OrgNotFoundError`, `InvalidRateError`.

## Database tables
- `organizations` (orm.py): id, name, commission_rate.

## Dependencies
- None on other modules. Pure relationship data.

## Test command
    pytest modules/organization/tests -q
```

## 2. Interface — `common/interfaces.py` (written before `service.py`)

```python
from decimal import Decimal
from typing import Protocol
from uuid import UUID


class OrganizationServiceInterface(Protocol):
    # The single, authoritative source of the commission rate.
    # If a value or method is not here, it is not part of this service.
    def get_commission_rate(self, org_id: UUID) -> Decimal: ...
    def set_commission_rate(self, org_id: UUID, rate: Decimal) -> None: ...


class NotificationServiceInterface(Protocol):
    def notify_assignment(self, entity_id: str, user_id: str) -> None: ...
    def notify_completion(self, entity_id: str) -> None: ...
```

Callers in other modules depend on the Protocol, never on the concrete class:

```python
class BillingService:
    def __init__(self, org_service: OrganizationServiceInterface) -> None:
        self._org_service = org_service  # injected at the composition root

    def platform_fee(self, org_id: UUID, amount: Decimal) -> Decimal:
        return amount * self._org_service.get_commission_rate(org_id)
```

## 3. Tests — written against the interface, MUST fail first (red phase)

The stub raises `NotImplementedError`, so these tests fail until step 4. Confirm the red
state — a test that passes before implementation is testing nothing. See the
`tests-as-ground-truth` skill and `superpowers:test-driven-development` for the mechanics.

```python
# modules/organization/service.py — stub written so tests import and FAIL
class OrganizationService:
    def get_commission_rate(self, org_id: UUID) -> Decimal:
        raise NotImplementedError
    def set_commission_rate(self, org_id: UUID, rate: Decimal) -> None:
        raise NotImplementedError
```

```python
# modules/organization/tests/test_service_organization.py
import pytest
from decimal import Decimal


def test_get_commission_rate_returns_stored_rate(org_service, seeded_org):
    assert org_service.get_commission_rate(seeded_org.id) == Decimal("0.15")


def test_get_commission_rate_raises_for_unknown_org(org_service):
    with pytest.raises(OrgNotFoundError):
        org_service.get_commission_rate(uuid4())


def test_set_commission_rate_rejects_negative(org_service, seeded_org):
    with pytest.raises(InvalidRateError):
        org_service.set_commission_rate(seeded_org.id, Decimal("-0.01"))
```

Run and confirm failure before writing real code:

```bash
pytest modules/organization/tests -q   # expect: failures, all NotImplementedError
```

## 4. Implementation — last

Only now does the agent fill in `service.py` (and the `orm.py` table) until the tests pass.
No new public methods appear that are not already in `OrganizationServiceInterface`; if one is
needed, it goes back to step 1 (update the README and the Protocol first).
