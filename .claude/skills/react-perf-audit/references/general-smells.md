# General Runtime Smells

> The commands below use `src/` as a placeholder for your actual audit root — replace it with the real path (`.` for a whole-repo audit, or the specific directory/files you were given) before running them.

Everything here shares one trait: it's a cost paid repeatedly (on every render, every scroll event, every mount) rather than once, which is what makes it worth finding even when each individual instance looks cheap in isolation.

## Expensive synchronous work in the render path

A computation that's O(n) or worse over a non-trivial dataset, run directly in a component body (not in `useMemo`), means it reruns on every render regardless of whether its inputs changed:

```bash
grep -rn "\.sort(\|\.filter(\|\.reduce(\|\.map(" src/ --include="*.tsx" | grep -v "useMemo\|useCallback"
```

Not every hit is a problem — sorting a 5-item array is free. Judge by the realistic size of the data and how often the component re-renders for unrelated reasons (check against what you already found in the re-renders category). The fix is `useMemo` keyed on the actual inputs, or moving the computation to where the data is fetched/transformed once rather than on every render.

## Un-throttled/un-debounced high-frequency handlers

Scroll, resize, mousemove, and keystroke-driven handlers (search-as-you-type, live validation) that do non-trivial work on every single event firing — these can fire dozens of times per second:

```bash
grep -rln "onScroll\|onMouseMove\|addEventListener.*scroll\|addEventListener.*resize" src/ --include="*.ts*"
grep -rn "onChange" src/ --include="*.tsx" -A 3 | grep -B 3 "fetch\|useQuery\|dispatch"
```

For each hit, check what the handler does. A handler that just sets simple local state is usually fine (React batches/is cheap here). A handler that triggers a network request, a heavy re-render, or expensive DOM reads needs throttling/debouncing — look for whether `lodash.debounce`/`throttle` or a custom hook is already applied; if not, that's the finding.

## Non-virtualized long lists

Rendering every item in a long list to the DOM at once, rather than only the visible window, costs both initial render time and ongoing scroll performance:

```bash
grep -rln "react-window\|react-virtualized\|@tanstack/react-virtual" package.json
grep -rn "\.map(" src/ --include="*.tsx" -B 3 | grep -B 3 "ul>\|<Table\|<tbody"
```

If a virtualization library is in `package.json`, check whether the specific list you're looking at actually uses it — it's common for one long list in a codebase to be virtualized (because someone hit the problem once) while others weren't converted. Flag proportional to realistic list length: a list of a user's own saved items (bounded, usually small) isn't worth virtualizing; a list of all conversations/records/rows in a shared dataset usually is.

## Memory leaks from missing cleanup

A `useEffect` that subscribes to something (an event listener, an interval, a websocket, an external store) without returning a cleanup function leaks that subscription every time the component unmounts and remounts — over a long session (especially in an SPA that never does a full page reload) this accumulates and causes real, hard-to-diagnose slowdown:

```bash
grep -rn "addEventListener\|setInterval\|subscribe(" src/ --include="*.tsx" -A 10 | grep -B 10 "useEffect"
```

For each `useEffect` containing one of these calls, confirm the effect returns a matching `removeEventListener`/`clearInterval`/`unsubscribe`. A missing cleanup here is a real bug, not just a style issue — rate it at least Medium, higher if the component mounts/unmounts frequently (a modal, a tab panel, a list item).

## Layout thrashing (forced synchronous reflow)

Reading a layout-dependent property (`offsetWidth`, `offsetHeight`, `getBoundingClientRect()`, `getComputedStyle()`) immediately after writing a style change, especially inside a loop, forces the browser to synchronously recalculate layout instead of batching it — doing this repeatedly (e.g. once per list item) can visibly stall the main thread:

```bash
grep -rn "getBoundingClientRect\|offsetWidth\|offsetHeight\|scrollHeight" src/ --include="*.ts*"
```

Check whether each read happens right after a write (a style/class change) to the same or a related element, especially inside `.map()` or a loop. The fix is to batch all the reads first, then all the writes, rather than interleaving them per element.

## Unoptimized images

Images are frequently the largest payload on a page and a common cause of both slow LCP and layout shift:

```bash
grep -rln "<img" src/ --include="*.tsx"
```

For each, check for: missing explicit `width`/`height` (or aspect-ratio CSS) causing CLS as the image loads; missing `loading="lazy"` on below-the-fold images; and whether the project has an image optimization pipeline (a framework's built-in `<Image>` component, a CDN with resize params) that's available but not being used at a given call site.
