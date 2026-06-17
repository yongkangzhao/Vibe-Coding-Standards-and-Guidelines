# Frontend: Tests as Ground Truth (vitest + Testing Library + MSW)

The frontend analog of the backend standards for a TypeScript / React stack. The "production engine" you must not substitute is **the real DOM and a real-ish network** — render the actual component, drive HTTP through MSW, and assert what the user sees.

## Layout — tests live beside the feature slice

Frontend code is organized into `src/features/<feature>/` slices (see
`architecture-and-module-boundaries`), and tests are **colocated** next to what they exercise — the
React analog of the backend's `test_service_<domain>` / `test_security_<name>` pytest split.

```
src/features/task/
├── TaskList.tsx
├── TaskList.test.tsx           # behavior test, colocated (one concern per file)
├── useTasks.ts
├── useTasks.test.ts
├── task.security.test.tsx      # exploit / authz tests (the test_security_* analog)
├── api.ts                      # typed client for the task endpoints
├── handlers.ts                 # shared MSW handlers (the "conftest" analog)
└── README.md
```

One concern per file; keep each under ~500 lines and split when it grows (pure refactor — same test count, same results).

## Render the real component, real-ish network via MSW

Do **not** shallow-render and do **not** mock the unit under test. Render into a real DOM with Testing Library; let MSW intercept the actual `fetch`/XHR so the component exercises its real data path.

```ts
// src/test/server.ts — shared across the suite (vitest setupFiles)
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);

// vitest.setup.ts
import { afterAll, afterEach, beforeAll } from "vitest";
import { server } from "./src/test/server";

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

```ts
// src/test/handlers.ts
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("/api/tasks", () =>
    HttpResponse.json({ items: [{ id: "t1", title: "Fix the bug", status: "open" }], total: 1 }),
  ),
];
```

## Assert observable behavior (what the user sees)

```tsx
// src/features/task/TaskList.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { http, HttpResponse } from "msw";
import { server } from "@/test/server";
import { TaskList } from "./TaskList";

test("renders tasks returned by the API", async () => {
  render(<TaskList userId="user-123" />);
  // Assert the rendered DOM, not internal state or props.
  expect(await screen.findByText("Fix the bug")).toBeInTheDocument();
});

test("shows an error when assignment targets a missing user", async () => {
  // Override the seam at the network boundary, not the component under test.
  server.use(
    http.post("/api/tasks/:id/assign", () =>
      HttpResponse.json({ code: "USER_NOT_FOUND" }, { status: 404 }),
    ),
  );
  render(<TaskList userId="user-123" />);
  await userEvent.click(await screen.findByRole("button", { name: /assign/i }));
  expect(await screen.findByRole("alert")).toHaveTextContent(/user not found/i);
});
```

## Mock across the seam — never the unit under test

The network is the module boundary (the backend is a neighbor service). MSW stubs that seam. The component, its hooks, and its real render tree all run for real.

```tsx
// FORBIDDEN: mocking the component or hook you are asserting on
vi.mock("./TaskList");           // <-- same false green as shallow render
vi.mock("./useTasks");           // <-- the unit under test must run

// LEGITIMATE: stub the network seam (the neighbor backend)
server.use(http.get("/api/tasks", () => HttpResponse.json({ items: [], total: 0 })));
```

Mock a genuinely external, non-deterministic collaborator (analytics SDK, clock) — but only when it sits across a real boundary, the way the backend mocks a neighbor service at its interface.

## Types are part of the contract

Keep the API client types aligned with the backend's response shape so a contract drift fails at compile time, not only at runtime:

```ts
// src/features/task/api.ts
export interface TaskResponse { id: string; title: string; status: "open" | "done" }
export interface TaskListResponse { items: TaskResponse[]; total: number }
```

## Run the module's tests in isolation

```bash
vitest run src/features/task
```

As on the backend, a post-edit hook can run exactly this and inject failures back into the agent's context — closing the feedback loop without a human.
