# Frontend: Verify Through The Real Path (TypeScript / React / vitest)

Concrete realization of the SKILL standards for a React stack. The throughline: a component test that mocks the network never sends the request that 500s, so it stays green while the deployed feature is broken for every user.

## Why a green component test hides a server-side 500

The component renders, fires the request, and shows data — but in the test the request is *mocked*, so the broken backend route is never hit. The same call against the real (or staging) backend returns 500.

```tsx
// ❌ Component test: the network is mocked, so the route that 500s is never exercised
vi.mock("../api", () => ({
  fetchInbox: vi.fn().mockResolvedValue([{ id: "1", subject: "hi" }]),  // green, and lying
}));

test("renders inbox", async () => {
  render(<Inbox />);
  expect(await screen.findByText("hi")).toBeInTheDocument();   // passes; prod is 500
});
```

Mocking the network is fine for asserting *render logic* — but it is not evidence the **route works**. Something has to drive the real request.

## ✅ A real-path integration test hits the actual endpoint contract

Drive the genuine request path. Either a thin API integration test against a running backend (best), or MSW configured to mirror the **real** response shape and status — including the failure the broken route would return.

```ts
// api.realpath.test.ts — runs against a real/staging API base, asserts the contract end-to-end
import { fetchInbox } from "../api";

test("GET /inbox returns 200 through the real route", async () => {
  const res = await fetchInbox({ baseUrl: process.env.STAGING_API! }); // real request, real router
  expect(res).toBeInstanceOf(Array);   // a 500 here fails loudly — the component test never could
});
```

```ts
// If you must use MSW, mirror reality — including the status the real server returns.
// ❌ Hand-wave a 200 the real route never gives:
http.get("/inbox", () => HttpResponse.json([]));
// ✅ Encode the actual contract so a regression in shape/status fails the test:
http.get("/inbox", () => HttpResponse.json({ messages: [] }, { status: 200 }));
```

## ✅ A live Playwright smoke through the deployed UI is the gate

`render()` + MSW proves the component; only clicking the genuine deployed UI proves the *system*. This is the check that surfaces a route that 500s for every user — make it a gate, not a manual afterthought.

```ts
// e2e/inbox.smoke.spec.ts — runs against the DEPLOYED app, real BFF/API behind it
import { test, expect } from "@playwright/test";

test("inbox loads for a signed-in user on the deployed site", async ({ page }) => {
  await page.goto(`${process.env.DEPLOY_URL}/inbox`);
  await signIn(page, process.env.SMOKE_USER!, process.env.SMOKE_PASS!);
  // Assert real content is in the DOM — not a spinner, not an error boundary.
  await expect(page.getByRole("heading", { name: "Inbox" })).toBeVisible();
  await expect(page.getByTestId("inbox-error")).toHaveCount(0);   // a 500 would trip this
});
```

## ✅ "Done" = a screenshot of the real route, not green tests

A feature is done when you have SEEN it in the deployed environment. Capture proof only after the page-specific DOM is present — a screenshot of a spinner proves nothing.

```ts
// ❌ Screenshot taken while the page is still loading — proves nothing
await page.goto(`${DEPLOY_URL}/inbox`);
await page.screenshot({ path: "proof.png" });          // might be a spinner or an error

// ✅ Wait for real, page-specific content, THEN capture
await page.getByText("hi").waitFor();                  // the actual data is rendered
await page.screenshot({ path: ".artifacts/inbox-proof.png" });
```

## ✅ Confirm the deploy actually rolled

A green CI run is not the new bundle serving. Cache-busting, a skipped build, or a CDN can leave stale assets live.

```ts
// Assert the running build, or assert the changed behavior directly (the old bug is gone).
test("deployed UI is serving the new build", async ({ page }) => {
  await page.goto(`${process.env.DEPLOY_URL}/`);
  const sha = await page.evaluate(() => (window as any).__BUILD_SHA__);
  expect(sha).toBe(process.env.EXPECTED_SHA);          // OR assert the fixed copy/behavior is present
});
```

## When a test contradicts the live bug — check the tree first

The component test is green, but the real UI errors.

```bash
# ❌ Wrong move: "tests pass, so the bug is imaginary / it must be a CDN or env issue"
# ✅ First: is the deployed bundle the code you're reading?
git status                                   # fix applied-but-uncommitted?
git diff origin/main -- src/Inbox.tsx        # local tree ahead of deployed?
curl -s $DEPLOY_URL/version.json | jq .sha   # deployed SHA == HEAD?
```

A green mocked test against a live 500 is the signature of a *seam-mocked* gap — not an imaginary bug. Reconcile the deployed SHA with HEAD before blaming infra; the broken request the mock skipped is real.
