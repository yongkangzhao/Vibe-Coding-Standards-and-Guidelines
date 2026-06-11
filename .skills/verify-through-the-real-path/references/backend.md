# Backend: Verify Through The Real Path (Python / FastAPI / SQLAlchemy / pytest)

Concrete realization of the SKILL standards for a FastAPI / pytest stack. The throughline: a test that calls the service method directly, or a test that mocks the request, both skip the router glue — where the 500 actually lives.

## The seam-mocked 500 — green tests, broken for every caller

A handler reads a field off the auth object that does not exist. The service is fine; the router line is not.

```python
# router.py — the model field is `user_id`, but the handler reads `.uid`
@router.get("/inbox")
def get_inbox(current_user: User = Depends(get_current_user), svc: InboxService = Depends()):
    return svc.list_for_user(current_user.uid)   # ❌ AttributeError → 500 for EVERY caller
```

Both of these stay green and prove nothing about the route:

```python
# ❌ service test — calls the method directly, never executes the broken router line
def test_list_for_user(inbox_service):
    msgs = inbox_service.list_for_user("user-123")   # green
    assert msgs == []

# ❌ "endpoint" test that over-mocks — patches the handler's collaborator, not the auth wiring
def test_inbox_route(monkeypatch):
    monkeypatch.setattr("router.InboxService.list_for_user", lambda self, uid: [])
    ...   # still never reads current_user.uid the way production does
```

## ✅ Drive the real route with production-shaped auth

Use `TestClient` so the request goes through the **real** router, the **real** auth dependency, and the **real** serializer. Override the auth dependency with an identity shaped exactly like production (same fields), not a loose stub.

```python
# tests/test_route_inbox.py
from fastapi.testclient import TestClient
from app.main import app
from app.auth import get_current_user
from app.models import User

def override_user():
    # Production-shaped: the real model, real field names. A dict/Mock here would hide the .uid bug.
    return User(user_id="user-123", email="a@b.co")

app.dependency_overrides[get_current_user] = override_user
client = TestClient(app)

def test_inbox_returns_200_through_the_real_router():
    resp = client.get("/inbox")
    assert resp.status_code == 200          # ❌ AttributeError → 500 here; this is the test that catches it
    assert resp.json() == []
```

If `get_current_user` were faked with a `Mock()`, `.uid` would resolve to a child mock and the bug would hide again. The override must use the **real type**.

## ✅ A live smoke through the deployed endpoint is the gate

`TestClient` catches the wiring bug pre-merge; a live smoke confirms the deployed environment actually serves it. Make this a gate, not a manual afterthought.

```python
# smoke/test_inbox_live.py  — run against the DEPLOYED base URL post-deploy
import os, httpx

def test_inbox_live_smoke():
    base = os.environ["DEPLOY_BASE_URL"]      # the real deployed env, not localhost
    token = os.environ["SMOKE_AUTH_TOKEN"]
    r = httpx.get(f"{base}/inbox", headers={"Authorization": f"Bearer {token}"}, timeout=10)
    assert r.status_code == 200               # would have been 500 in prod for days
```

## ✅ Confirm the roll — a green build is not the new code serving

Expose the running SHA and assert it post-deploy. A build that succeeds proves an artifact exists, not that it answers requests.

```python
# app/main.py
import os
@app.get("/version")
def version():
    return {"sha": os.environ.get("GIT_SHA", "unknown")}
```

```python
# smoke/test_version.py — run after deploy with the SHA you just built
import os, httpx
def test_deployed_sha_matches_build():
    r = httpx.get(f'{os.environ["DEPLOY_BASE_URL"]}/version', timeout=10)
    assert r.json()["sha"] == os.environ["EXPECTED_SHA"]   # OR assert the old bug is gone
```

## ✅ Harden the deploy; rebuild on every deployable change

```yaml
# ❌ image build gated on a path filter, but the deploy points at the new SHA → stale image, fresh tag
build-image:
  if: contains(github.event.head_commit.modified, 'Dockerfile')   # skips most commits

# ✅ rebuild on every change to deployable code; retry the transient roll; verify it landed
deploy:
  steps:
    - run: docker build -t app:${{ github.sha }} . && docker push app:${{ github.sha }}
    - run: |
        for i in 1 2 3; do kubectl rollout status deploy/app --timeout=120s && break; sleep 10; done
    - run: test "$(curl -s $DEPLOY_BASE_URL/version | jq -r .sha)" = "${{ github.sha }}"
```

## When grep contradicts the live bug — check the tree first

A route 500s in the deployed env, but `grep` shows the handler reads `.user_id` and looks correct.

```bash
# ❌ Wrong move: conclude "the code is fine, must be DNS / the load balancer / a caching layer"
# ✅ First: is what's running what you're reading?
git status                       # fix applied-but-uncommitted?
git diff origin/main -- router.py  # local tree differs from what deployed?
curl -s $DEPLOY_BASE_URL/version   # deployed SHA == HEAD?
```

Nine times out of ten the deployed SHA is behind HEAD, or the fix is staged-not-pushed. Don't invent an infra cause for a code bug (or a code cause for an infra bug) until the SHAs line up and `git diff` is clean.
