# Backend: Frontend Honesty (Python / FastAPI / SQLAlchemy / pytest)

The backend can't draw a pixel, but it decides what's *true*. The honesty failures on
screen almost always trace to a response that erased the difference between "absent" and
"zero", leaked an identifier, or returned the wrong status code so the client couldn't
tell error from empty. Send the truth shaped so the client can render it honestly.

## Absent must stay absent — don't backfill null with 0

```python
# ❌ Coalescing null to 0 in the response — the client now can't tell "unknown" from "zero"
class IncomeResponse(BaseModel):
    monthly_total: Decimal = Decimal("0")   # default hides "no data yet"

def get_income(self, coach_id: UUID) -> IncomeResponse:
    row = self._db.scalar(select(func.sum(PayoutORM.amount)).where(...))
    return IncomeResponse(monthly_total=row or Decimal("0"))   # ← lie: None became $0.00

# ✅ Optional field; None means "no measurement", a real 0 means "measured zero"
class IncomeResponse(BaseModel):
    monthly_total: Decimal | None    # None → client renders "—"; 0 → "$0.00"

def get_income(self, coach_id: UUID) -> IncomeResponse:
    row = self._db.scalar(select(func.sum(PayoutORM.amount)).where(...))
    return IncomeResponse(monthly_total=row)   # None stays None across the wire
```

`COUNT` returns 0 for no rows (a true zero); `SUM`/`AVG`/`MAX` return `NULL` (no measurement).
Preserve that distinction — don't `or 0` it away in the service or the schema.

## Serialize decimals as strings, never floats

```python
# ✅ Decimal serialized as a string preserves precision; client narrows it before math
class PriceResponse(BaseModel):
    amount: Decimal          # FastAPI/Pydantic emits "19.99" (string), not 19.99 (float)
    model_config = ConfigDict(json_encoders={Decimal: str})
```

Floats lie quietly — `0.1 + 0.2` is `0.30000000000000004`. A money field crosses the wire
as a string so the client can choose how to parse it; the client must narrow it with a typed
helper (see `references/frontend.md`), not run arithmetic on the raw field.

## Return real status codes — error must not look like empty

```python
# ❌ Swallowing the failure into a 200 with an empty list — client renders "No items yet"
@router.get("/inbox")
def inbox(user_id: str) -> list[MessageResponse]:
    try:
        return self._svc.list_messages(user_id)
    except Exception:
        return []          # ← a 500's worth of failure now reads as a calm empty inbox

# ✅ Let the real status through; missing resource is 404, failure is 5xx, empty is 200 []
@router.get("/inbox")
def inbox(user_id: str) -> list[MessageResponse]:
    return self._svc.list_messages(user_id)   # genuinely-empty → 200 []; broken → 5xx

@router.get("/coaches/{coach_id}/profile")
def profile(coach_id: UUID) -> CoachProfileResponse:
    profile = self._svc.get_profile(coach_id)
    if profile is None:
        raise HTTPException(status_code=404, detail="profile_not_found")  # not 200 {}
    return profile
```

A 200 with `[]` means "really empty"; a 5xx means "broken". Collapsing the second into the
first is the server half of the "200-lie" — the client literally cannot render error vs empty
honestly if both arrive as `200 []`.

## Never leak identifiers in error detail

```python
# ❌ Identifier bytes in the error string — these end up echoed into the DOM
raise HTTPException(404, detail=f"No coach profile for user_id='{user_id}'")

# ✅ Stable machine code + safe message; the raw id stays in server logs only
logger.warning("coach profile missing", extra={"user_id": user_id})
raise HTTPException(status_code=404, detail="coach_profile_not_found")
```

`detail` is a contract the client maps to copy, not a debugging scratchpad. Put the `user_id`
in structured logs; send a code the frontend can translate to honest, generic text.

## Don't fabricate stats — return null/empty, let the client show a placeholder

```python
# ❌ Inventing a rating when none has been computed yet
return CoachCardResponse(name=name, rating=4.8)   # there is no rating source — fabricated

# ✅ Optional; None means "no rating yet", and the client renders a placeholder not a number
class CoachCardResponse(BaseModel):
    name: str
    rating: float | None      # None until real reviews exist; never a made-up default
```

## Test the honesty boundary

```python
def test_income_none_is_not_zero():
    # no payouts exist → SUM is NULL → response must carry None, not Decimal("0")
    resp = client.get(f"/coaches/{coach_id}/income").json()
    assert resp["monthly_total"] is None        # client will render "—", not "$0.00"

def test_missing_profile_is_404_not_empty_200():
    resp = client.get(f"/coaches/{unknown_id}/profile")
    assert resp.status_code == 404              # client tells error from empty
    assert "user_id" not in resp.text           # no identifier leaked in detail
```
