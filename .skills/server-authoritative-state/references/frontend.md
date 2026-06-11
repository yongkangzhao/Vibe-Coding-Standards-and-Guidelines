# Frontend: Server-Authoritative State (TypeScript / React / TanStack Query)

The client is a cache. Every pattern here exists to stop the cache from asserting something the server hasn't confirmed. APIs shown are TanStack Query v5 style.

## Reflect the server's returned result — don't flip the UI to the assumed outcome

The failure mode is a phantom success: the optimistic flip shows "Saved," the request fails, and the change vanishes on reload. Await the server and render what it returns.

```tsx
// ❌ Flips to the assumed outcome; a failure leaves a green "Saved" that's a lie
const [saved, setSaved] = useState(false);
async function onSave(input: Patch) {
  setSaved(true);                 // phantom success — server hasn't agreed
  await api.patch(`/courts/${id}`, input);   // if this throws, UI still says "Saved"
}

// ✅ Await the truth; render the returned object; on failure leave state unchanged + show the error
const mutation = useMutation({
  mutationFn: (input: Patch) => api.patch<Court>(`/courts/${id}`, input),
  onSuccess: (court) => queryClient.setQueryData(["court", id], court),  // server's row
  onError: (err) => toast.error(getMessage(err)),                        // honest failure
});
```

> Optimistic updates are not banned — but if you use one, you MUST roll it back on error (snapshot in `onMutate`, restore in `onError`; see `data-and-state-safety`). An optimistic flip with no rollback is the phantom-success bug. When in doubt, prefer return-the-result over optimistic.

## Write the cache from the response — never hand-edit the list

Your local guess at the new state drifts from the server on the first edge case (a server default, a trigger, a normalized field). Overwrite with what came back, or invalidate.

```ts
// ❌ Local guess: assumes you know the post-mutation shape — you don't
queryClient.setQueryData<Court[]>(["courts"], (list) =>
  list?.map((c) => (c.id === id ? { ...c, ...patch } : c)),  // drifts from server truth
);

// ✅ Overwrite the entry with the server's object, then reconcile the list
onSuccess: (court) => {
  queryClient.setQueryData(["court", court.id], court);      // detail = server truth
  queryClient.invalidateQueries({ queryKey: ["courts"] });   // list refetches truth
},
```

## Re-entrancy: a ref guard, not just disabled state

`disabled` and a state flag both lag a fast double-click by a render, so the mutation fires twice. A ref flips synchronously and wins the race.

```ts
// ❌ State-only guard: `submitting` hasn't re-rendered yet on the second click → fires twice
const [submitting, setSubmitting] = useState(false);
async function onClick() {
  if (submitting) return;          // stale on the 2nd synchronous click
  setSubmitting(true);
  await api.post("/checkout", body);   // can run twice → double charge
}

// ✅ Synchronous ref guard (+ disabled for UX). The ref is set before the next click can read it.
const inFlight = useRef(false);
async function onClick() {
  if (inFlight.current) return;
  inFlight.current = true;
  try {
    await api.post("/checkout", body, { headers: { "Idempotency-Key": keyRef.current } });
  } finally {
    inFlight.current = false;
  }
}
// <button disabled={mutation.isPending} onClick={onClick}>  // disabled is the UX, ref is the guard
```

## A confirmed delete must not be resurrected by a stale list

A successful-but-stale refetch can re-include a just-removed item, and an auto-select can re-arm a removed payment method that then reaches the charge call. Filter the removed id out until the server's list agrees.

```tsx
// ✅ Track ids the server has confirmed removed; never render or auto-select them
const removed = useRef(new Set<string>());

const del = useMutation({
  mutationFn: (id: string) => api.delete(`/payment-methods/${id}`),
  onSuccess: (_d, id) => {
    removed.current.add(id);                                  // server confirmed
    queryClient.invalidateQueries({ queryKey: ["payment-methods"] });
  },
});

const methods = (data ?? []).filter((m) => !removed.current.has(m.id));  // lagging refetch can't re-list it

// ❌ Auto-select re-arms a removed card → it reaches the charge call as a phantom default
useEffect(() => { if (!selected) setSelected(methods[0]?.id); }, [methods]);
// ✅ Only auto-select something the server still lists AND we didn't just remove
useEffect(() => {
  if (selected && removed.current.has(selected)) setSelected(undefined);
  if (!selected) setSelected(methods.find((m) => !removed.current.has(m.id))?.id);
}, [methods, selected]);
```

## No client-computed authority

Permissions, prices, availability, and membership status come from the response. Computing them client-side guarantees the UI eventually disagrees with the server's real decision.

```tsx
// ❌ Client-derived price and permission — both drift from what the server actually does
const price = basePriceCents * (1 - (isMember ? 0.1 : 0));     // server owns discounts
{user.id === court.ownerId && <EditButton />}                  // server owns the permission

// ✅ Render the authority the server returned; the client only displays it
<Price cents={booking.priceCents} />                           // the price they WILL be charged
{booking.canEdit && <EditButton />}                            // the server's decision
{booking.canCancel ? <CancelButton /> : null}
```

If a needed field (`priceCents`, `canEdit`, `availableSlots`) isn't in the response, the fix is a server contract that returns it — file the endpoint, don't compute it here.

## Replace-all editors send the loaded version and handle 409

The client can't enforce optimistic locking, but it must participate: send the version it loaded and treat a 409 as "this changed under you — reload," not a generic error.

```tsx
const save = useMutation({
  mutationFn: (slots: Slot[]) =>
    api.put(`/courts/${id}/availability`, { version: loadedVersion, slots }),
  onSuccess: (fresh) => queryClient.setQueryData(["availability", id], fresh),
  onError: (err) => {
    if (isConflict(err) /* 409 */) {
      toast.warn("Availability changed since you loaded it — reloading the latest.");
      queryClient.invalidateQueries({ queryKey: ["availability", id] });  // re-base the editor
    } else {
      toast.error(getMessage(err));
    }
  },
});
```
