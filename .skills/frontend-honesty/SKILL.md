---
name: frontend-honesty
description: Forbids the UI from ever displaying something untrue — fabricated zeros for missing data, a failed fetch shown as an empty list, a raw identifier shown as a name, or a number coerced from garbage. Use this when building or reviewing any view that renders a value, money, count, rating, or status; when writing a number/money/percent formatter; when handling loading/empty/error states; when mapping a backend error to on-screen copy; or whenever an agent is about to render a plausible-looking value with no real data behind it — for both backend (what the API returns) and frontend (what the screen shows), even if the user doesn't say "honesty."
---

# Frontend Honesty

> The UI must never display something untrue. A lie in the UI is worse than a crash: a crash is loud and gets fixed; a lie looks fine and ships. This matters most when an agent writes the rendering code — an agent will happily render `$0.00` for a value the API didn't send, turn a 500 into "No items yet", and print a raw `user_id` where a name belongs. Every one of those is plausible, passes a glance, and is false. Honesty is the rule that the pixels on screen must correspond to a true fact the backend actually returned.

## When to use this
- Building or reviewing any view that renders a value, money amount, count, rating, percentage, or status.
- Writing or touching a formatter — `formatMoney`, `formatPercent`, a number/date helper, a name resolver.
- Handling the loading / empty / error states of a fetch (especially "what do we show when it fails?").
- Mapping a backend error or exception to on-screen copy, or rendering anything derived from a server response.
- A value can be `null`/absent and the code is about to substitute `0`, `$0.00`, `0%`, or `NaN`.
- A list/empty-state is keyed on `!data` — and a failed fetch ALSO produces no data.
- An identifier (`uuid`, `user_id`, an order key) is anywhere near visible text, a label, or a name slot.
- Any number on screen (a stat, a rating, a delta) and you can't point at the exact backend field behind it.

## The standards

**Absent is not zero.** A missing or null value renders an honest placeholder — an em-dash, "Not set", "—" — NEVER a fabricated `0`, `$0.00`, `0%`, or `NaN`. Zero is a real measured value; absence is the lack of one, and conflating them tells the user a balance is empty when it's actually unknown. Formatters must distinguish the two at the boundary.
```
format_money(null)  -> "—"      # absence
format_money(0)     -> "$0.00"  # a real, measured zero
```

**Error is not empty.** A failed fetch and a genuinely-empty result are different facts and must render differently. Never key the empty-state on "no data" alone, because an error also yields no data — gate it on an explicit error flag from the data layer, not on the truthiness of an error message string. Error message strings are an unreliable signal: some transport failures carry an empty message, so `if (errorMessage)` silently falls through to the empty branch and a 500 renders as "Nothing here yet" — the 200-lie.
```
if (isError)      render the error state      # explicit flag, not message truthiness
else if (isEmpty) render the empty state
else              render the data
```

**Never render internal identifiers, and never echo a raw backend error.** A `uuid` or `user_id` is bytes for joins, not text for humans — and a truncated id prefix is still id bytes. A raw server error string ("No profile for user_id='tB4...'") leaks those bytes straight into the DOM. Resolve identifiers to a human-meaningful name, or show an honest placeholder; map backend errors to honest, generic copy. Never pass an exception message through to the screen verbatim.

**Gate every numeric formatter through a strict check before coercion.** `Number()` / `parseFloat` will cheerfully turn `"1e3"` into `1000`, `"0x14"` into `20`, and `""` into `0` — each a fabricated number presented as truth. Validate the input against a strict decimal shape FIRST; anything that doesn't match degrades to the honest placeholder, it does not coerce. Money that crosses the wire as a string for decimal precision must be narrowed by a typed helper before any arithmetic — never do math on the raw field.
```
if not matches(value, /^-?\d+(\.\d+)?$/):  return "—"   # reject "1e3", "0x14", "", "NaN"
return format(to_number(value))
```

**Loading, empty, and error are first-class states, rendered independently per card.** Each must be an explicit, separate branch — not an afterthought fallthrough. And one card's failed fetch must not blank the page: isolate fetches so a broken income widget shows its own error tile while the rest of the dashboard renders. A single failure that whites out the whole screen is its own kind of lie (the page looks "empty", not "partly broken").

**Every number traces to a real backend field, or it is removed.** No fabricated stats, ratings, counts, or deltas — "432 players", "★4.8", "+12.5%" with no endpoint behind them are forbidden, even as placeholder polish. If the data isn't available, show a placeholder or omit the element; never invent a plausible figure to fill the space. A made-up number is indistinguishable from a real one to the user, which is exactly why it's dangerous.

## Checklist
- [ ] Null/absent values render an em-dash or honest placeholder, never a fabricated `0` / `$0.00` / `0%` / `NaN`.
- [ ] Empty-state is gated on an explicit `isEmpty` AND error-state on an explicit `isError` flag — not on `!data` or message-string truthiness.
- [ ] No raw `uuid` / `user_id` / identifier is rendered as visible text or as a name (not even a truncated prefix).
- [ ] No backend error/exception string is shown verbatim — all are mapped to honest, generic copy.
- [ ] Every numeric formatter rejects non-decimal input (`"1e3"`, `"0x14"`, `""`) before `Number()`, degrading to a placeholder.
- [ ] Loading / empty / error are separate rendered branches; one card's failure doesn't blank the page.
- [ ] Every visible number, rating, and count points at a specific backend field — no invented figures.

## What breaks without this
- **A balance of "unknown" shows as `$0.00`.** The user trusts an empty wallet that may be full; absence was rendered as a measured zero.
- **A 500 reads as "No items yet."** The empty-state keyed on `!data` (or on a message string that was empty) hid a server outage behind a calm, false "all caught up."
- **Identifier bytes leak into the DOM.** A raw `user_id` or an echoed error message ("No profile for user_id='tB4...'") ships PII-adjacent internal data to the screen.
- **Garbage coerces to a confident number.** `"1e3"` became `100000%`, `""` became `$0` — `Number()` manufactured a precise-looking lie because nothing validated the input first.
- **One broken widget whites out the whole page.** A single failed fetch with no card isolation turns a partly-broken dashboard into a blank one — which the user reads as "nothing's here."

## Stack-specific examples
- **Backend:** see `references/backend.md` — what the API must (and must not) send: `null` not `0`, real error codes not leaked identifiers, decimal-as-string serialization, proper status codes so the client can tell error from empty.
- **Frontend:** see `references/frontend.md` — react-query `isError`/`isEmpty` branching, strict `formatMoney`/`formatPercent`, name resolution, per-card error isolation, no fabricated stats.

## Related
- `verify-through-the-real-path` — honest states are only honest if the failure paths are actually exercised; that skill makes you prove the error/empty branches against the real request.
- `data-and-state-safety` (frontend) — the mutation-side rules (optimistic rollback, idempotency, server-side authz); this skill is the read/render side.
- `interface-first-development` — a typed API contract from the backend is what lets the client tell "null" from "0" and "error" from "empty" in the first place.
