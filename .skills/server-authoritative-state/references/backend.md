# Backend: Server-Authoritative State (Python / FastAPI / SQLAlchemy / pytest)

The server is the source of truth, so it must make the client's job easy: return the persisted state, enforce the preconditions the client can't, and make deletes stay deleted. APIs shown are SQLAlchemy 2.x / FastAPI style.

## A mutation returns the persisted resource — not 204, not the input

The client reflects what you return. If you return nothing (or echo the request body), the client is forced to guess the new state, and its guess is the bug.

```python
# ❌ Returns 204; client has to invent the new state and patch its cache by hand
@router.patch("/courts/{court_id}", status_code=204)
def update_court(court_id: UUID, patch: CourtUpdate, db: Session = Depends(get_db)) -> None:
    court = _get(db, court_id)
    for k, v in patch.model_dump(exclude_unset=True).items():
        setattr(court, k, v)
    db.flush()
    return None  # client now guesses; its guess drifts from server-applied defaults/triggers

# ✅ Return the persisted row so the client overwrites its cache with truth
@router.patch("/courts/{court_id}", response_model=CourtResponse)
def update_court(court_id: UUID, patch: CourtUpdate, db: Session = Depends(get_db)) -> CourtResponse:
    court = _get(db, court_id)
    for k, v in patch.model_dump(exclude_unset=True).items():
        setattr(court, k, v)
    db.flush()
    db.refresh(court)              # pick up DB-computed columns (updated_at, server defaults)
    return CourtResponse.model_validate(court)
```

## Authority is computed and returned by the server

The client renders permissions, prices, and availability; it must never derive them. Put the decision in the response so the client has nothing to recompute.

```python
# ✅ The response carries the authority the UI needs — no client-side derivation
class BookingResponse(BaseModel):
    id: UUID
    price_cents: int            # the price the user WILL be charged, computed here
    can_cancel: bool            # the server's permission decision, not user.id == owner_id
    available_slots: list[Slot] # availability the server computed, not a client rule

def get_booking(booking_id: UUID, caller: UUID, db: Session) -> BookingResponse:
    booking = _get(db, booking_id)
    return BookingResponse(
        id=booking.id,
        price_cents=pricing.quote(booking),                 # one source of price truth
        can_cancel=perms.can(caller, booking.facility_id, "booking:cancel"),
        available_slots=calendar.open_slots(booking.court_id),
    )
```

If the client is tempted to compute one of these (a discount, an "is owner" check, a slot rule), that is the signal to add the field here — not to compute it client-side.

## Replace-all writes need an optimistic-locking precondition

A stale editor's `PUT` (loaded an hour ago) silently overwrites a newer save. Reject any write whose base version doesn't match current — the client sends the version it loaded.

```python
class AvailabilityReplace(BaseModel):
    version: int                # the version the client loaded
    slots: list[SlotInput]

def replace_availability(court_id: UUID, body: AvailabilityReplace, db: Session) -> AvailabilityResponse:
    # ❌ No precondition: a stale editor's slots clobber a fresher save and nobody notices.
    # ✅ Atomic compare-and-set: only bump version if it still matches what the client loaded.
    bumped = db.scalar(
        update(AvailabilityORM)
        .where(AvailabilityORM.court_id == court_id)
        .where(AvailabilityORM.version == body.version)     # the precondition
        .values(version=AvailabilityORM.version + 1)
        .returning(AvailabilityORM.version)
    )
    if bumped is None:
        # current version != loaded version → someone saved in between
        raise HTTPException(status_code=409, detail="stale_version")  # client: reload + retry
    _replace_slots(db, court_id, body.slots)
    db.flush()
    return _load(db, court_id)
```

The `.where(version == loaded)` guard *is* the lock — it is enforced in one atomic UPDATE, not a read-then-write that a concurrent request can interleave.

## A confirmed delete stays deleted — and reads never resurrect it

Eventual consistency means a list query that ran a moment before the delete can still return the row. The fix is on the read side: filter so a removed entity can't be re-listed (this is also why deletes that matter are soft — see `data-and-state-safety`).

```python
# ✅ Every list excludes removed rows; a lagging client refetch can't resurrect them
def list_payment_methods(user_id: UUID, db: Session) -> list[PaymentMethodResponse]:
    rows = db.scalars(
        select(PaymentMethodORM)
        .where(PaymentMethodORM.user_id == user_id)
        .where(PaymentMethodORM.deleted_at.is_(None))    # removed → never returned
        .order_by(PaymentMethodORM.created_at.desc(), PaymentMethodORM.id)
    ).all()
    return [PaymentMethodResponse.model_validate(r) for r in rows]

# ✅ And the charge path re-validates against truth — a detached/removed card must not charge
def charge(self, user_id: UUID, method_id: UUID, amount: int, db: Session) -> Charge:
    method = db.scalar(
        select(PaymentMethodORM)
        .where(PaymentMethodORM.id == method_id)
        .where(PaymentMethodORM.user_id == user_id)
        .where(PaymentMethodORM.deleted_at.is_(None))
    )
    if method is None:
        raise PaymentMethodGoneError()   # a stale client reference must fail, not charge
    ...
```

## Idempotency makes a double-fired mutation safe

The client guards re-entrancy, but a retry still reaches you. Collapse duplicates server-side so a second arrival is a no-op that returns the first result.

```python
def create_booking(self, key: str, req: BookingCreate, db: Session) -> BookingResponse:
    existing = db.scalar(select(BookingORM).where(BookingORM.idempotency_key == key))
    if existing is not None:
        return BookingResponse.model_validate(existing)   # replay, don't double-book
    booking = BookingORM(idempotency_key=key, **req.model_dump())
    db.add(booking)
    db.flush()   # UNIQUE(idempotency_key) is the real guard under a race
    return BookingResponse.model_validate(booking)
```

## Testing the contract

```python
def test_failed_update_leaves_state_unchanged(db, court):
    with pytest.raises(ForbiddenError):
        svc.update_court(court.facility_id, court.id, caller="intruder", name="hacked")
    assert _get(db, court.id).name == court.name           # server state never moved

def test_stale_replace_all_is_rejected(db, court):
    v0 = svc.get_availability(court.id).version
    svc.replace_availability(court.id, AvailabilityReplace(version=v0, slots=[...]))  # bumps to v1
    with pytest.raises(HTTPException) as e:                 # second editor still on v0
        svc.replace_availability(court.id, AvailabilityReplace(version=v0, slots=[...]))
    assert e.value.status_code == 409                       # newer save not silently clobbered
```
