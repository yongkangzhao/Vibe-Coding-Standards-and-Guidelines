# Frontend: Clean Migration (TypeScript / React / vitest)

Concrete patterns for the SKILL.md standards. The recurring frontend failure is the duplicated component: three near-identical copies, one gets a fix, the others rot.

## Promote a triplicated component to one canonical copy

Three screens each have their own copy of a payment-method form trio. Promote one copy to the shared package, migrate all three consumers, and delete the duplicates in the same change.

```
# ❌ Before — three drifting copies, each slightly different
apps/checkout/components/CardForm.tsx
apps/billing/components/CardForm.tsx
apps/settings/components/CardForm.tsx

# ✅ After — one canonical copy, all consumers import it
packages/shared/src/payments/CardForm.tsx
```

```ts
// ❌ A consumer still importing its local copy — not migrated
import { CardForm } from "./components/CardForm";

// ✅ Every consumer points at the one shared copy; locals are deleted
import { CardForm } from "@shared/payments";
```

Don't leave one screen on its old local copy "because it's slightly customized" — fold the variation into a prop on the shared copy, or it's still three components.

## Forbid a fourth copy with a structural test

A migration that only deletes duplicates lets the next agent re-create one. Make re-introduction a test failure.

```ts
import { globSync } from "glob";
import { describe, it, expect } from "vitest";

describe("CardForm has exactly one canonical copy", () => {
  it("is not duplicated outside the shared package", () => {
    const copies = globSync("**/CardForm.tsx", {
      ignore: ["**/node_modules/**", "packages/shared/src/payments/CardForm.tsx"],
    });
    expect(copies).toEqual([]);   // any other CardForm.tsx fails CI
  });
});
```

This is the structural equivalent of the backend "old module must not import" test — the duplicate can't quietly come back.

## Rename a route — migrate consumers, no redirect-forever shim

A concept moved from `/wallet` to `/transactions` + `/memberships`. A permanent redirect keeps the dead route — and the dead mental model — alive. Update every link and delete the old route.

```tsx
// ❌ Redirect shim left in the router "so old links work" — forever
<Route path="/wallet" element={<Navigate to="/transactions" replace />} />

// ✅ Old route deleted; every Link migrated in this change
<Route path="/transactions" element={<Transactions />} />
<Route path="/memberships" element={<Memberships />} />
```

```bash
rg -n '/wallet' apps/        # zero hits — no Link, no redirect, no test still on it
```

A real "users have old bookmarks" need is a *server* 301 with a removal date and an issue — not a client-side `<Navigate>` that lives in the router forever.

## Rename a type — migrate all fields, delete the shadow

When a hand-written type is replaced by a generated/canonical one, migrate every field in the batch. A shadow type that coexists lets two definitions of the same shape drift apart.

```ts
// ❌ Shadow type kept alongside the generated one — they will diverge
interface Booking { id: string; price: number; }        // hand-written, stale
import type { BookingResponse } from "@shared/types/generated"; // canonical

// ✅ One source of truth; the hand-written shadow is deleted, all sites migrated
import type { BookingResponse } from "@shared/types/generated";
```

Accept that this touches many files at once — that batch is the migration. A `// TODO: migrate later` on the old type is how you end up with two `Booking`s a year from now.
