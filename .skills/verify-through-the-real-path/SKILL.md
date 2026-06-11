---
name: verify-through-the-real-path
description: Insists that "it works" be proven through the actual request path, the deployed artifact, and the genuine UI — not inferred from green unit tests that mocked the exact seam that breaks. Use this when deciding whether a feature is done, reviewing a change that "passes all tests," wiring or trusting CI/CD, mocking the network or auth in a test, debugging a live bug that grep says shouldn't exist, or choosing what evidence counts as proof — for both backend and frontend, even if the user doesn't say "verify through the real path."
---

# Verify Through The Real Path

> Green unit tests prove the unit, not the system. A feature can be 100% green and 100% broken in production because every test mocked the exact seam that fails — the router that wires the handler, the auth object the handler reads, the deployed image that actually serves. An agent optimizes for the signal it's given: if green tests are the only gate, it will write code that passes them and never executes the broken glue. The fix is to make the real path the gate — drive the actual HTTP request, observe the deployed environment, click the genuine UI — so "done" means *observed working*, not *plausibly correct*.

## When to use this
- A feature's unit and component tests are green and you're about to call it done.
- A router/controller reads off an auth or context object (`current_user.<field>`) that no test ever exercises through the real route.
- Reviewing a PR where the service-layer test and the component test both pass but nothing drove a request end-to-end.
- Wiring or trusting CI/CD: a build job went green and you're assuming the new code is now serving.
- A deploy step is flaky, gated on a path filter, or you can't say which SHA is live.
- A live-observed bug contradicts a grep or a passing test ("that line looks fine, this can't be happening").
- Deciding what counts as proof: a screenshot, a log line, a live journey assertion — or just "tests pass."
- Closing out anything user-facing: the bar is "I have SEEN it work in the deployed environment."

## The standards

**Seam-mocked tests skip the glue; drive the real route too.** The most dangerous bug is the one every test routes around. A handler that reads the wrong field off the auth object throws on its first real line — yet a component test that mocks the network passes (the request never reaches the handler) and a service test that calls the method directly passes (the broken router line is never executed). Both are green; every real caller gets a 500. For every route, add one test that drives the **real** path — an in-process request through the actual router, with auth shaped like production — plus a live smoke through the genuine UI.
```
# unit test: calls the method directly — never touches the router line that breaks
service.handle(fake_request)            # green, and lying

# real-path test: goes through the router, the auth wiring, the serializer
response = client.GET("/inbox", as_user=real_identity)
assert response.status == 200            # this is the one that catches the 500
```

**"Done" includes deployed-and-observed.** A feature is not done when tests pass; it is done when you have SEEN it work where it actually runs. Visual proof — a screenshot of the real route at the real viewport, or a live journey assertion against the deployed environment — is a **gate, not a nicety**. "Tests are green" and "I observed it working" are different claims; only the second closes the task.
```
gate(feature):
    require(tests_green)                 # necessary
    require(observed_in_deployed_env)    # and sufficient only WITH this
```

**A build succeeding is not the new code serving.** A green pipeline means an artifact was produced, not that it's the artifact answering requests. After deploy, confirm the **roll**: assert the live artifact's version/SHA, or assert the changed behavior directly (the old bug is provably gone, not "should be gone"). Rebuild on *every* deployable change — never gate the image build behind a path filter while the deploy points at the new SHA, or you ship a stale image under a fresh tag. Harden transient deploy steps with retry; a flaky deploy that silently leaves old code running is a regression no test will catch.
```
deploy():
    build(); push()
    rollout_with_retry()
    assert live_version() == built_sha   # OR assert old_bug_is_gone() against live
```

**When grep contradicts a live bug, check the working tree before inventing a cause.** A live-observed failure that "can't exist" per the source usually means the source you're reading isn't the source that's running: the fix is applied-but-uncommitted, the deployed SHA differs from HEAD, or a stale build is serving. Look at `git diff`/`git status` and the deployed SHA **first**. Don't manufacture an infra explanation for a code bug, or a code explanation for an infra bug, without evidence — the evidence is one `git diff` and one version probe away.

## Checklist
- [ ] Every route has a real-path test that goes through the router with production-shaped auth — not just a direct service-method call.
- [ ] At least one live smoke drives the genuine UI / deployed endpoint for the changed behavior.
- [ ] "Done" is backed by an observation in the deployed environment (screenshot or live journey assertion), not only green tests.
- [ ] Post-deploy, the live artifact's SHA/version is asserted, or the changed behavior is confirmed live (old bug gone).
- [ ] The image rebuilds on every deployable change; no path filter can leave the deploy pointing at a stale SHA.
- [ ] Transient deploy steps retry; a failed roll fails loud, not silently leaving old code.
- [ ] A live bug that contradicts the source was reconciled against `git diff` + deployed SHA before any infra theory.

## What breaks without this
- **A 500 for every user, green the whole time.** A handler reads the wrong auth field; the component test mocked the network, the service test bypassed the router, so the broken line never ran in any test — and it 500s for every real caller, for days, until someone clicks the real UI.
- **"Shipped" code that never shipped.** The build is green, the PR is merged, but a path filter skipped the image rebuild — the deploy serves a stale image and the "fix" was never live.
- **Hours lost to an imaginary infra incident.** A live bug "can't exist" per grep, so the search moves to load balancers and DNS — when one `git diff` shows the fix was committed but never deployed.
- **Flaky-deploy regressions.** A transient deploy step fails silently, old code keeps serving, every test still passes, and the regression is invisible until a user reports it.

## Stack-specific examples
- **Backend:** see `references/backend.md` — a `TestClient` request through the real router with production-shaped auth, the `current_user.uid` vs `user_id` 500, asserting the deployed SHA, retry on the deploy step.
- **Frontend:** see `references/frontend.md` — why a mocked-network component test stays green while the route 500s, a real-path API/integration test, and a live Playwright smoke through the deployed UI as the gate.

## Related
- `tests-as-ground-truth` — run the real engine, not a substitute; this skill extends that from the storage seam to the **request/deploy/UI** seam.
- `adversarial-exploit-review` — the independent reviewer turns "should work" into a failing test driven through the real path before the claim is believed.
- `frontend-honesty` — the honest error/empty states this skill insists you exercise are only honest if a real failed request was actually observed hitting them.
