# Network & Data-Fetching Smells (including N+1)

> The commands below use `src/` as a placeholder for your actual audit root — replace it with the real path (`.` for a whole-repo audit, or the specific directory/files you were given) before running them. A command that silently matches nothing because the placeholder path doesn't exist will under-report an entire category as clean.

This is the frontend mirror of a backend N+1 query: instead of one query fetching N rows, you get N network round-trips because each row/item in a list independently fetches its own data. The backend may be perfectly efficient and this still tanks the page, because the bottleneck is N round-trip latencies in the browser, not server-side query cost. On a high-latency connection (mobile, VPN, a slow region) this is often the single biggest perceived-performance problem on a page, bigger than any re-render issue — prioritize finding it.

## N+1 fetch waterfalls

**The pattern:** a parent renders a list, and each list item component independently calls a data-fetching hook for its own data, instead of the parent fetching all the data in one batched request and passing it down (or the items sharing one query keyed on the full set of IDs).

```bash
# Find list rendering...
grep -rn "\.map(" src/ --include="*.tsx" -A 2

# ...then check whether the mapped component calls a query/fetch hook itself
grep -rln "useQuery\|useSWR\|useFetch\|useEffect.*fetch" src/ --include="*.tsx"
```

Cross-reference: if a component appears both (a) rendered via `.map()` in a parent and (b) contains its own `useQuery`/`fetch`/`useEffect` data call keyed on a prop like `id`, that's an N+1 candidate. Confirm by checking the query key — `['item', id]` fetched once per rendered item is the signature. The fix is almost always one of:

- A single query at the parent keyed on the full list of IDs (`['items', ids]`), fetched once, with results distributed to children as props.
- A backend endpoint that already supports batch-fetching by ID list, if `grep`-ing the API client reveals one exists but isn't being used.
- If there's genuinely no batch endpoint, that's worth surfacing as a finding pointing at the API layer, not just the component — the real fix may not be in this codebase at all.

This exact pattern also shows up outside list rendering: a parent component fetching entity A, then a child fetching entity B using a field from A, when B's fetch doesn't actually need to wait on A resolving and both could be kicked off in parallel. Look for a `useQuery` whose `enabled` depends on another query's data where the dependency is only used to pick an ID that was actually available earlier (e.g. from a route param, not from the first query's response).

## Sequential awaits that should run in parallel

```bash
grep -rn "await " src/ --include="*.ts*" -A 1 | grep -B 1 "await "
```

Two or more consecutive `await` calls with no data dependency between them (the second doesn't use the first's result) should be `Promise.all`'d. This is cheap to spot and easy to fix, and directly adds up latency on every affected code path:

```ts
// Before - serial, ~2x latency
const user = await fetchUser(id)
const settings = await fetchSettings(id)

// After - parallel
const [user, settings] = await Promise.all([fetchUser(id), fetchSettings(id)])
```

## Missing caching / request deduplication

If the project uses a data-fetching library with a cache (TanStack Query, SWR, Apollo, RTK Query), check for places that bypass it with a raw `fetch`/`axios` call inside a `useEffect` instead of going through the library's hook — those requests get neither caching, deduplication of concurrent identical requests, nor automatic retry/invalidation:

```bash
grep -rn "useEffect" src/ --include="*.tsx" -A 5 | grep -B 5 "fetch(\|axios\."
```

Also check for the same query being fetched with inconsistent keys in different places (`['user', id]` in one file, `['users', id]` in another) — this defeats deduplication even when the library is used correctly, because the cache treats them as unrelated.

## Overfetching

A GraphQL query requesting fields the component doesn't render, or a REST call hitting an endpoint that returns a large object when a narrower one exists (or query params to narrow it exist and aren't used). This matters most on mobile/slow connections and on frequently-refetched queries — a field nobody reads that happens to be a large embedded array or blob is worth flagging even if nothing else looks wrong.

## Missing pagination / unbounded fetches

A list endpoint called with no limit, or a "load everything then filter/paginate client-side" pattern. Flag this proportional to how large the underlying collection can realistically grow — a config list capped at 20 by the data model isn't a finding; a list of conversations, users, or any user-generated collection is.

## Polling and subscriptions

Short polling intervals (sub-5s) on data that doesn't need that freshness, or a `setInterval`/polling query that isn't cleaned up when the component unmounts (see `references/general-smells.md` for the cleanup-leak pattern generally — this is the data-fetching-specific case of it). Prefer: a longer interval, a server push mechanism if one exists, or `refetchOnWindowFocus`-style revalidation instead of a fixed timer.
