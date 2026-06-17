# Frontend: Feature Slices in TypeScript/React

The same standards in a TypeScript / React stack. Backend has domain *service* modules; the frontend
has feature/domain *slices*. Organize by feature, expose a public barrel, enforce import boundaries
with lint, and keep business logic out of components.

## Feature-sliced layout

Group by feature, not by technical type. A `global components/ + hooks/ + utils/` soup gives every
change a wide blast radius; feature folders keep it narrow — and uniform, so one feature teaches all.

```
src/
├── features/
│   ├── task/
│   │   ├── index.ts            # PUBLIC barrel — the only entry point
│   │   ├── api.ts              # data fetching for this feature
│   │   ├── model.ts            # business logic / state (the "service")
│   │   ├── components/         # feature-private UI
│   │   └── task.test.ts
│   └── notification/
│       └── index.ts
├── shared/                     # cross-feature primitives only (Button, fetcher, types)
└── app/
    └── providers.tsx           # composition root — wires features + dependencies
```

```ts
// features/task/index.ts — the entire public surface of the feature
export { TaskList } from "./components/TaskList";
export { useCompleteTask } from "./model";
export type { TaskId } from "./types";
// api.ts and internal components are NOT exported. No feature reaches into them.
```

## No cross-feature imports

`features/notification` may not import from `features/task/model`. It imports the public barrel, or
nothing. Shared concerns live in `shared/`. This is the seam — keep it clean so a feature can move
to its own bundle or micro-frontend later.

```ts
// features/notification/model.ts
import { type TaskId } from "@/features/task";   // OK — public barrel
// import { internalThing } from "@/features/task/model";  // ✗ blocked by lint
```

## Interfaces / types before implementation

The contract is a TS type, written before the component. A method not in the type is not part of the
feature. (The interface-first-development skill owns *how* to drive these.)

```ts
// shared/contracts.ts — written first
export type TaskId = string & { readonly __brand: "TaskId" };  // opaque id across the seam

export interface NotificationService {
  notifyCompletion(entityId: TaskId): Promise<void>;
}
```

Pass opaque branded IDs across feature boundaries, never another feature's record/ORM shape — no
coupling to unpick.

## Keep business logic out of components

Components are a thin presentation layer. Validation, authorization, and pricing belong to the
backend; the feature `model`/`api` only orchestrates calls. If the backend endpoint is missing, file
an issue — don't reimplement the rule client-side.

```ts
// features/task/model.ts — orchestration, no business rules
export function useCompleteTask(notifier: NotificationService) {
  return useMutation({
    mutationFn: (id: TaskId) => api.completeTask(id),   // backend decides "can complete?"
    onSuccess: (_, id) => notifier.notifyCompletion(id),
  });
}
```

## The composition root

One place wires dependencies into features — swap an implementation here, callers stay on the type.

```tsx
// app/providers.tsx — the ONE wiring location
const notifier: NotificationService = new HttpNotificationService(env.API_URL);

export function AppProviders({ children }: PropsWithChildren) {
  return <NotifierContext.Provider value={notifier}>{children}</NotifierContext.Provider>;
}
```

## Enforce the boundary structurally

A guideline relies on memory; ESLint rejects the import. Use `eslint-plugin-boundaries` or
`no-restricted-imports` so an agent literally cannot wire two features together.

```jsonc
// .eslintrc — block reaching into feature internals
"no-restricted-imports": ["error", {
  "patterns": [{
    "group": ["@/features/*/!(index)", "@/features/*/*"],
    "message": "Import the feature's public index barrel, not its internals."
  }]
}]
```

## Tests as ground truth

vitest + Testing Library for behavior, MSW to stub the backend at the network layer (never mock the
feature's own internals), and validate boundary payloads with zod.

```ts
// features/task/task.test.ts
import { render, screen } from "@testing-library/react";
import { http, HttpResponse } from "msw";
import { server } from "@/test/server";

it("notifies on completion", async () => {
  server.use(http.post("/tasks/:id/complete", () => HttpResponse.json({ status: "done" })));
  const notifier = { notifyCompletion: vi.fn() };
  render(<TaskList notifier={notifier} />);
  await screen.findByText("done");
  expect(notifier.notifyCompletion).toHaveBeenCalledWith("task-123");
});
```
