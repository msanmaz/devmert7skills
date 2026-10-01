# Unnecessary Re-renders

> The commands below use `src/` as a placeholder for your actual audit root — replace it with the real path (`.` for a whole-repo audit, or the specific directory/files you were given) before running them.

The mental model: a component re-renders when its parent re-renders, its own state changes, or a context it subscribes to changes. None of that is inherently bad — React is designed to re-render cheaply. It becomes a *performance* problem specifically when a re-render does expensive work (a large subtree, an expensive computation, a big list) that didn't need to happen because the inputs didn't meaningfully change. Chase that distinction, not "did this re-render" in isolation — a component re-rendering 10 times a second is irrelevant if it's a `<span>`.

## What to search for

**Inline object/array/function literals passed as props to a memoized child.** This is the single most common way `React.memo` silently stops working — a new object identity every render defeats the memo's shallow comparison.

```bash
grep -rn "style={{" src/ --include="*.tsx"
grep -rn "onClick={() =>" src/ --include="*.tsx"
grep -rn "onChange={() =>" src/ --include="*.tsx"
```

For each hit, check whether the receiving component is wrapped in `React.memo`. If it isn't, this is a non-issue — flag it only when it's actually defeating a memoization that exists, or when the child is expensive enough (a large list, a chart) that wrapping it *and* stabilizing the prop would be worth doing.

**`useEffect`/`useMemo`/`useCallback` with a missing or over-broad dependency array**, specifically where the effect itself causes a state update that triggers another render — look for the "effect calls setState" pattern, which is a common source of double-renders and cascading updates:

```bash
grep -rn "useEffect(" src/ --include="*.tsx" -A 5
```

Read each hit's dependency array against what it actually reads. An object or array literal in the deps array (`[{ id }]`, `[items.filter(...)]`) recreates every render and defeats the effect's own memoization — same bug as the props case above.

**Context providers wrapping large subtrees with an unmemoized value**, so every consumer re-renders on every provider re-render regardless of whether the piece of state they read changed:

```bash
grep -rln "createContext" src/ --include="*.ts*"
```

For each context, find its `<Context.Provider value={...}>`. If `value` is an inline object literal (`value={{ foo, bar }}`) rather than a `useMemo`'d value, every re-render of the provider component creates a new context value and re-renders every consumer — count how many components consume it (`useContext(ThisContext)`) to judge blast radius.

**List rendering with an unstable or missing `key`.** `key={index}` on a list that can reorder, filter, or have items inserted/removed from the middle causes React to misattribute state and re-mount/re-render more than necessary (and can cause real bugs, not just perf ones). This is exactly what a lint rule should already catch (`eslint-plugin-react`'s `react/no-array-index-key`) — check for it first per the "not a linter" guidance in SKILL.md. If it isn't enforced, search for the literal pattern directly rather than trying to associate it with a specific `.map()` via a line-count context window, which breaks on any multi-line JSX:

```bash
grep -rn "key={index}" src/ --include="*.tsx"
grep -rn "key={i}" src/ --include="*.tsx"
```

**Large components that re-render entirely when only a small piece of their output actually changed.** Look for a component that holds both frequently-changing state (e.g. an input value, a hover state) and renders an expensive child (a table, a chart) in the same return — the fix is usually to lift the volatile state into a smaller child component rather than memoizing the expensive one.

## What a good fix looks like

- Wrap the unstable value in `useMemo`/`useCallback` **only if** something downstream actually benefits (a memoized child, a dependency array elsewhere). Memoizing a value nothing checks for referential equality is pure overhead — flag that as a non-fix if you see it already done pointlessly.
- Memoize the context value with `useMemo`, or split one large context into several smaller ones scoped to what actually changes together.
- Move volatile local state down into a leaf component instead of memoizing everything above it.
- Replace `key={index}` with a stable identifier from the data (`key={item.id}`).

Note which React version/compiler setup the project uses before flagging any of this — if the React Compiler (React 19+ `babel-plugin-react-compiler` or equivalent) is enabled, most manual `useMemo`/`useCallback`/`React.memo` work is done automatically and flagging their absence is noise. Check `package.json`/babel or vite config for the compiler plugin first.
