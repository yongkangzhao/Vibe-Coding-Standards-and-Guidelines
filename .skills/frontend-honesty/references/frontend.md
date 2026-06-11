# Frontend: Frontend Honesty (TypeScript / React / vitest)

This is where the lie actually reaches the user. The patterns below are the recurring,
hard-won ones: a null coerced to `$0.00`, a 500 rendered as an empty list, a raw id shown
as a name, garbage coerced to a confident number. Each has a one-line fix that an agent
will skip unless the rule is explicit.

## Absent is not zero — formatters return a placeholder for null

```ts
// ❌ Renders a fabricated zero for missing data
export function formatMoney(value: number | null): string {
  return `$${(value ?? 0).toFixed(2)}`; // null → "$0.00" — a lie about an unknown balance
}

// ✅ Absence → placeholder; a real 0 → "$0.00"
export function formatMoney(value: number | null | undefined): string {
  if (value == null) return "—";          // unknown is not zero
  return `$${value.toFixed(2)}`;
}
```
```tsx
<dd>{formatMoney(account.balance)}</dd>   {/* "—" when the API sent null, "$0.00" only for a true 0 */}
```

## Error is not empty — branch on isError, never on !data or a message string

```tsx
// ❌ A failed fetch yields no data, so this renders the empty state for a 500 (the "200-lie")
function Inbox() {
  const { data, error } = useQuery({ queryKey: ["inbox"], queryFn: fetchInbox });
  if (error?.message) return <ErrorTile />;     // ← some 500s have an EMPTY message → falls through
  if (!data?.length) return <EmptyState text="No messages yet" />; // shown for the failed fetch
  return <MessageList items={data} />;
}

// ✅ Explicit isError flag from the data layer, checked before empty
function Inbox() {
  const { data, isLoading, isError } = useQuery({ queryKey: ["inbox"], queryFn: fetchInbox });
  if (isLoading) return <InboxSkeleton />;
  if (isError) return <ErrorTile onRetry={() => /* refetch */} />;  // failure has its own state
  if (data.length === 0) return <EmptyState text="No messages yet" />; // only a real empty
  return <MessageList items={data} />;
}
```

Never gate on `error.message` truthiness: an HTTP/2 500 carries an empty `statusText`, so
`error.message` can be `""`, and `if (error.message)` silently renders the empty state for a
hard failure. Use the boolean `isError`.

## Never render an identifier — resolve to a name or a placeholder

```tsx
// ❌ Raw user_id (or a truncated prefix) shown where a human name belongs — id bytes in the DOM
<span>{message.senderId}</span>                 {/* "tB4f9a2c-..." */}
<span>{message.senderId.slice(0, 8)}</span>     {/* still id bytes, just fewer */}

// ✅ Resolve to a real name; fall back to honest copy, never to the id
<span>{message.senderName ?? "Unknown sender"}</span>
```

## Never echo a backend error verbatim — map it to honest copy

```tsx
// ❌ Server detail leaks identifier bytes straight onto the screen
catch (err) { setError(err.response.data.detail); } // "No profile for user_id='tB4...'"

// ✅ Map the machine code to safe, generic copy
const MESSAGES: Record<string, string> = {
  coach_profile_not_found: "This coach hasn't set up a profile yet.",
};
catch (err) {
  const code = err.response?.data?.detail;
  setError(MESSAGES[code] ?? "Something went wrong. Please try again.");
}
```

## Strict numeric formatters — validate before Number()

```ts
const DECIMAL = /^-?\d+(\.\d+)?$/;

// ❌ Number() coerces garbage into confident lies
export function formatPercent(v: string): string {
  return `${Number(v) * 100}%`;   // "1e3" → 100000%, "0x14" → 2000%, "" → 0%
}

// ✅ Reject anything that isn't a plain decimal, degrade to a placeholder
export function formatPercent(v: string | number | null): string {
  const s = String(v ?? "");
  if (!DECIMAL.test(s)) return "—";       // "1e3", "0x14", "", "NaN" → honest dash
  return `${(Number(s) * 100).toFixed(1)}%`;
}
```

Decimal money arrives as `number | string` (serialized as a string for precision). Narrow it
through one typed helper before any arithmetic — never `+`/`*` a raw decimal field, JS will
coerce and lie about both the value and the type.

```ts
export function parseDecimal(v: string | number | null): number | null {
  const s = String(v ?? "");
  return DECIMAL.test(s) ? Number(s) : null;   // null, not 0, when it isn't a clean decimal
}
```

## Card isolation — one failed fetch must not blank the page

```tsx
// ✅ Each card owns its own query + its own loading/empty/error; a failure stays local
function Dashboard() {
  return (
    <div className="grid grid-cols-2 gap-4">
      <IncomeCard />   {/* if its fetch 500s, it shows its own ErrorTile... */}
      <BookingsCard /> {/* ...and these still render their real data */}
      <RosterCard />
    </div>
  );
}
```
Don't hoist every fetch into one top-level query whose error white-screens the whole page.

## No fabricated stats — every number traces to a real field

```tsx
// ❌ Invented figures with no endpoint behind them
<Stat label="Players" value="432" />
<Stat label="Rating" value="★4.8" />

// ✅ Real field, with an honest placeholder when absent
<Stat label="Players" value={facility.playerCount ?? "—"} />
{facility.rating != null && <Stat label="Rating" value={`★${facility.rating.toFixed(1)}`} />}
```

## Test the honesty boundary

```ts
import { render, screen } from "@testing-library/react";

it("renders a dash for a null balance, not $0.00", () => {
  expect(formatMoney(null)).toBe("—");
  expect(formatMoney(0)).toBe("$0.00");   // a real zero is still shown
});

it("rejects non-decimal input instead of coercing", () => {
  expect(formatPercent("1e3")).toBe("—");
  expect(formatPercent("0x14")).toBe("—");
  expect(formatPercent("")).toBe("—");
});

it("shows the error state on a failed fetch, not the empty state", async () => {
  server.use(http.get("/inbox", () => HttpResponse.json(null, { status: 500 })));
  render(<Inbox />);
  expect(await screen.findByRole("alert")).toBeInTheDocument();        // error tile
  expect(screen.queryByText("No messages yet")).not.toBeInTheDocument(); // NOT empty
});
```
