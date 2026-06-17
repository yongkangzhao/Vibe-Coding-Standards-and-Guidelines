# Frontend: Data & State Safety (TypeScript / React)

This standard is mostly a backend/persistence concern. The frontend has only a few real, distinct responsibilities — they are listed here in full. Do not invent more; soft-delete, atomic counters, savepoints, and migrations are the server's job.

## Authorization belongs on the server, not the client

Client-side checks are UX hints — they hide a button, nothing more. A user can call the API directly, so every authorization decision must be enforced server-side. Never gate a mutation on client state alone.

```tsx
// ✅ Hide the control for UX, but the SERVER must re-check on the request.
{post.canDelete && <button onClick={() => deletePost(post.id)}>Delete</button>}

// ❌ Do NOT compute permission from client data and treat it as authoritative:
if (currentUser.id === post.authorId) {
  await api.post(`/posts/${post.id}/force-delete`); // server must verify ownership too
}
```
If the backend has no endpoint that enforces the rule, file a GitHub issue — do not move the logic client-side.

## Idempotency keys for mutations

A double-click, a retry after a flaky network, or React StrictMode can fire the same mutation twice. Send a stable idempotency key so the server collapses duplicates into one effect.

```ts
async function purchase(cartId: string): Promise<Receipt> {
  const idempotencyKey = crypto.randomUUID(); // one key per user intent, reused on retry
  return api.post(
    "/checkout",
    { cartId },
    { headers: { "Idempotency-Key": idempotencyKey } },
  );
}
```
Generate the key once when the user commits the action and reuse it across retries — not a fresh key per attempt.

## Roll back optimistic updates on failure

Optimistic UI updates assume success. When the request fails, you must restore the previous state or the UI diverges from the server. With TanStack Query, snapshot in `onMutate`, restore in `onError`.

```ts
useMutation({
  mutationFn: (id: string) => api.post(`/posts/${id}/like`),
  onMutate: async (id) => {
    await queryClient.cancelQueries({ queryKey: ["post", id] });
    const previous = queryClient.getQueryData(["post", id]);     // snapshot
    queryClient.setQueryData(["post", id], (p: Post) => ({
      ...p, likeCount: p.likeCount + 1,
    }));
    return { previous };
  },
  onError: (_err, id, ctx) => {
    queryClient.setQueryData(["post", id], ctx?.previous);        // roll back
  },
  onSettled: (_d, _e, id) =>
    queryClient.invalidateQueries({ queryKey: ["post", id] }),    // reconcile with server
});
```

## Avoid lost updates in shared client state

When two tabs or async handlers write the same store, a stale read overwrites a fresh write — the frontend analog of the non-atomic counter. Update from the latest state (functional updater), don't compute from a captured snapshot.

```ts
// ❌ Captures a stale `count`; concurrent handlers clobber each other
setCount(count + 1);

// ✅ Functional update reads the latest committed value
setCount((c) => c + 1);
```
For state shared across tabs, treat the server as the source of truth and reconcile on focus/refetch rather than trusting local copies.
