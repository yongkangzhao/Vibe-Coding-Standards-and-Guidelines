---
name: server-authoritative-state
description: Treats the client as a cache of the server's truth — mutations reflect the server's returned result (never an assumed outcome), the cache is written from confirmed responses, and authority (permissions, prices, availability, membership) comes from the server, not client derivation. Use this when writing or reviewing an optimistic update, a mutation handler, a delete/remove-from-list flow, a refetch/invalidation, a double-submit guard, a replace-all editor, or any place the UI shows derived state — and when designing the endpoints that back them — for both backend and frontend, even if the user doesn't say "server-authoritative."
---

# Server-Authoritative State

> The client is a cache of the server's truth, not a second source of truth. The moment the UI asserts an outcome the server hasn't confirmed, you've created two competing versions of reality — and the client's version is the one users see, the one that survives a failed request, and the one that lies. This bites hardest when an agent writes the mutation: it reaches for the optimistic flip because it reads as "responsive," not noticing that a failed call now shows a phantom success that vanishes on reload. Make the server's response the only thing that moves the UI, and the lie becomes impossible.

## When to use this
- Wiring a mutation that creates, updates, or deletes a server resource (any `useMutation`, form submit, or action handler).
- About to optimistically flip the UI — toggle a like, mark "Saved," remove a row — before the server confirms.
- Updating a cached list/detail after a mutation: deciding between `setQueryData`, invalidate, or a local array edit.
- A button can be double-clicked or a mutation re-fired (StrictMode, retry, fast clicks): guarding re-entrancy.
- A confirmed delete that a stale refetch could re-list, or an auto-select that re-arms a just-removed item.
- Rendering anything authoritative — permission, price, availability, membership tier, quota — that the UI must not compute itself.
- Designing a "replace-all" / bulk-save editor (availability grid, settings panel) where two editors can clobber each other.
- Reviewing a PR where the UI "looks right" but you can't prove the server agreed.

## The standards

**No optimistic mutation that can mask a failure.** Mutate → call the server → reflect the server's **returned** result (or refetch). Never flip the UI to the assumed outcome before the server confirms, and never leave it flipped if the call fails. A failed mutation must leave the UI unchanged and surface an honest error — a green "Saved" that disappears on reload is data-loss theater.
```
onSubmit:  result = await server.save(input)   # await the truth
           render(result)                       # reflect what came back
           # on failure: state unchanged + visible error — never a phantom success
```

**The cache reflects server truth.** On a confirmed change, write the server's returned object into the cache (`setQueryData`) and/or invalidate + refetch. Don't hand-edit the local list as if you already know the new state — your guess and the server's reality drift on the first edge case.
```
saved = await server.update(id, patch)
cache.set([resource, id], saved)        # not: cache.map(x => x.id===id ? {...x, ...patch} : x)
```

**Guard every mutation against double-fire.** Disable the control while in-flight **and** add a synchronous ref guard. Component state lags a fast double-click by a render, so a state-only guard still fires twice; a ref flips immediately.
```
if (inFlightRef.current) return
inFlightRef.current = true            # synchronous — wins the race a setState loses
try { await server.mutate() } finally { inFlightRef.current = false }
```

**A confirmed delete is server truth; don't let a lagging list resurrect it.** A stale-but-successful refetch can re-include a just-removed item, and an auto-select can re-arm a removed payment method. After a confirmed delete, filter the removed id out of any rendered list until the server agrees, and never auto-select an id you just deleted.
```
visible = list.filter(x => !removedIds.has(x.id))   # eventual consistency: hide until server catches up
```

**No client-computed authority.** Permissions, prices, membership status, and availability come **from** the server; the client renders them, it never derives them. A price recomputed in JS, a permission inferred from `user.id === owner.id`, an availability computed from a local rule — each drifts from the server's actual decision. If the endpoint doesn't return what you need, add it server-side.
```
render(response.priceCents)            # not: basePrice * (1 - clientGuessedDiscount)
{response.canEdit && <EditButton/>}    # not: {user.id === resource.ownerId && ...}
```

**Replace-all writes need a concurrency precondition — on the server.** A stale editor's `PUT /availability` (loaded an hour ago) silently overwrites newer state with old data. The server must reject a write whose base version is stale via an `If-Match` / version token; the client sends the version it loaded and surfaces the 409 as "reload — this changed." Optimistic locking is a server guarantee; the client cannot enforce it.
```
PUT { version: loadedVersion, items: [...] }   # server: if current_version != loadedVersion → 409
```

## Checklist
- [ ] No mutation flips the UI before the server confirms; the UI reflects the **returned** result or a refetch.
- [ ] A failed mutation leaves state unchanged and shows a real error — never a phantom success.
- [ ] Cache updates use the server's returned object (`setQueryData`) or invalidate+refetch — not a local guess at the new state.
- [ ] Every mutation has a synchronous re-entrancy guard (ref) **plus** a disabled-while-in-flight control.
- [ ] A confirmed delete filters the id out of stale lists; no auto-select re-arms a removed item.
- [ ] No permission, price, availability, or membership status is computed client-side; all come from the response.
- [ ] Replace-all / bulk-save endpoints enforce an `If-Match`/version precondition server-side; the client sends its loaded version and handles 409.

## What breaks without this
- **Phantom success.** An optimistic flip shows "Saved" / "Added," the request actually failed, and the change vanishes on reload — the user believes data is safe that was never persisted.
- **Cache lies.** A hand-edited local list diverges from the server's real state on the first edge case the client's guess didn't model; the UI shows a row that doesn't exist (or hides one that does).
- **Double-effect.** A fast double-click fires the mutation twice — two charges, two rows, two emails — because the only guard was component state that hadn't re-rendered yet.
- **Resurrection.** A confirmed delete reappears because a stale refetch re-listed it (or auto-select re-armed a removed payment method that then reaches a charge call) — a detached object survives the failure and acts.
- **Authority drift.** A client-computed price, permission, or availability disagrees with the server's real decision — the user is shown a price they won't be charged, or an action they can't actually perform.
- **Lost update.** A stale editor's replace-all `PUT` silently wipes a newer save because nothing checked the base version.

## Stack-specific examples
- **Backend:** see `references/backend.md` (FastAPI/SQLAlchemy — mutations return the persisted resource, idempotency, `If-Match`/version optimistic locking, deletes that stay deleted).
- **Frontend:** see `references/frontend.md` (React/TanStack Query — return-result over optimistic flip, `setQueryData` from the response, ref re-entrancy guard, removed-id filtering, no client-derived authority).

## Related
- `data-and-state-safety` owns the persistence-side rules (soft-delete, atomic counters, savepoints, idempotency keys, optimistic-rollback mechanics) — this skill adds the "client is a cache, server is truth" framing and the authority/concurrency rules. Don't duplicate its rollback example.
- `interface-first-development` — if the client needs a value it's tempted to derive (price, permission), the fix is a server contract that returns it; define that interface first.
- `tests-as-ground-truth` — assert the failure modes here (failed mutation leaves UI unchanged, stale `PUT` gets 409) against the real engine, not a mock that always succeeds.
