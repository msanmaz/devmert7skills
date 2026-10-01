---
name: react-perf-audit
description: >-
  Audits a React / modern JS-TS frontend for performance problems — unnecessary
  re-renders, missing memoization, bundle-size bloat, missing code-splitting,
  Core Web Vitals (LCP/CLS/INP) regressions, and general main-thread/render-path
  smells — and produces a single prioritized findings report with file:line and
  a concrete fix per issue. Read-only: it reports, it does not edit code.
  Use this whenever someone asks to "audit performance", "check for slow
  renders", "why does this feel janky", "find re-render issues", "check bundle
  size", "run a Lighthouse/perf check", "review this PR for performance",
  or wants a performance pass on a React component, page, or whole frontend
  codebase — even if they just say "is this slow" or "can we make this faster"
  without naming a specific technique.
argument-hint: "[files, dir, PR#, or 'diff'] [--live <dev-server-url>]"
allowed-tools: Read, Glob, Grep, Bash(git *), Bash(npm *), Bash(pnpm *), Bash(yarn *), TodoWrite, MCPSearch, Write
---

# React Performance Audit

You are auditing a React/frontend JS-TS codebase for performance problems. Your output is **one findings report** — you diagnose and prioritize, you don't fix. If the user wants fixes applied, they'll ask for that as a separate step once they've seen what you found; mixing editing into an audit makes it harder to review what changed and why.

Performance problems compound invisibly: a component that re-renders 40 times instead of 4 doesn't crash anything, it just quietly burns CPU and battery, and nobody notices until the page already feels sluggish. That's why an explicit audit pass is worth doing even when nothing is "broken" — you're looking for things that work correctly but cost more than they need to.

## Scope: what are you auditing?

Figure this out before reading any code, the same way you'd scope any review:

- **A diff or PR** (`git diff`, a PR number, or "review this PR") — audit only the changed files, but read enough of the surrounding component tree to tell whether a change makes an *existing* re-render problem worse (e.g. a new prop that breaks memoization on a component that renders a long list).
- **Explicit files or a directory** — audit exactly what's named.
- **Nothing specified** — default to `git diff` against the repo's default branch (same convention as reviewing a PR before it ships). If that's empty, ask what to audit rather than guessing at a whole monorepo.

Use `TodoWrite` to track the categories below as you work through them — it keeps you from quietly skipping one on a large audit. If `TodoWrite` isn't available in your current environment (some subagent/task harnesses don't expose it), just track the categories yourself — it's a convenience, not a requirement.

The grep commands in the reference files use `src/` as a placeholder for your actual audit root — swap it for the real path (`.` for a whole-repo audit, or whatever directory/files you were actually given) before running them. A command that silently matches nothing because the placeholder path doesn't exist will make you report an entire category as clean when you simply didn't search it.

## The five things you're looking for

Each category below has a dedicated reference file with the specific code patterns, how to search for them, and what a good fix looks like. Read the reference file for a category before auditing it — don't rely on memory for the grep patterns, they're easy to get subtly wrong (e.g. an inline arrow function is only a problem if it's passed to a memoized child).

1. **Unnecessary re-renders** — `references/re-renders.md`. Missing memoization, unstable props/callbacks, context over-subscription, list key misuse.
2. **Network & data-fetching smells, including N+1** — `references/network-data-fetching.md`. A list that fires one request per item instead of one batched request, sequential awaits that should be parallel, missing cache/dedup, overfetching, missing pagination. This is the frontend equivalent of a backend N+1 query and is usually the single most impactful category on data-heavy pages — don't skip it.
3. **Bundle size** — `references/bundle-size.md`. Large/duplicate dependencies, missing code-splitting, barrel-file over-importing, dead exports.
4. **Core Web Vitals (live measurement)** — `references/web-vitals.md`. LCP/CLS/INP via the chrome-devtools MCP, only when a dev server is reachable. Skip this category gracefully if nothing is running — don't block the rest of the audit on it.
5. **General runtime smells** — `references/general-smells.md`. Expensive synchronous work in render, un-throttled handlers, non-virtualized long lists, memory leaks, layout thrashing, unoptimized images.

Work through all five unless the user's scope obviously excludes one (e.g. a pure presentational component with no data fetching has no N+1 story, a tiny 3-component diff doesn't need a Lighthouse run).

## Severity: how to rank a finding

Think about it the way you'd think about any defect — likelihood × blast radius, not "does this violate a rule":

- **Critical** — measurably blocks interaction or makes a Core Web Vital fail its threshold (see `references/web-vitals.md`) on a commonly-hit path.
- **High** — a hot path (frequently-mounted component, large list, high-traffic page) doing clearly avoidable repeated work — a re-render storm, a large eagerly-loaded chunk that could be split.
- **Medium** — correct but costly in a path that's not obviously hot, or a pattern that will become a problem as data grows (e.g. an unvirtualized list that's fine at 20 items today).
- **Low** — technically wasteful but immaterial at current scale (a `useCallback` that would help only if the child were memoized, and it isn't).

A finding you can't attach a plausible real-world cost to isn't a finding — it's a style opinion. If you catch yourself about to flag "this *could* be memoized" with no story for why it matters, that's a sign to either find the story (how often does this re-render, what does it cost) or drop it.

## Producing the report

Always use this structure. One report, findings ordered Critical → High → Medium → Low, ties broken by file path. Return it directly in your response by default — don't write it to a file. Only save it to disk (with `Write`) if the user explicitly asks for a saved copy; if they don't say where, default to `PERF_AUDIT.md` at the root of whatever you audited.

````markdown
# React Performance Audit

**Scope:** [diff / files / PR# / directory audited]
**Live measurement:** [ran against <url> | skipped — no dev server available]

## Summary
[2-3 sentences: overall health, the single biggest win if there's one]

## Findings

### [Severity] Short title
- **Category:** re-render | network-fetching | bundle-size | web-vitals | runtime
- **Location:** `path/to/file.tsx:42`
- **What's happening:** [concrete description of the mechanism, not just "this is slow"]
- **Why it matters:** [the cost — re-renders per interaction, KB added, ms of LCP, etc. Use real numbers from the trace/analysis where you have them; say "estimated" when you don't]
- **Fix:**
  ```tsx
  // concrete before/after or the specific change, not a vague suggestion
  ```

[repeat per finding]

## Not flagged (and why)
[Optional — things that look suspicious at a glance but you ruled out, e.g. "FeatureList.tsx remaps on every render but renders a 3-item array, not worth memoizing." This saves the next person from re-investigating the same thing.]
````

If you find genuinely nothing in a category, say so in the summary rather than omitting it silently — "no bundle-size issues found" is a useful, confidence-building result, not a gap in the report.

## What this skill is not

- Not a linter — don't flag anything your project's ESLint/`react-hooks` rules would already catch at build time. Check first:
  ```bash
  grep -n "react-hooks\|react-compiler\|react/no-array-index-key" .eslintrc* eslint.config.* 2>/dev/null
  ```
  If a rule already covers it, skip the finding — repeating what the linter already enforces is noise, not value.
- Not a full QA pass — functional correctness, accessibility, and visual regressions are out of scope here even if you notice them in passing (mention briefly, don't deep-dive).
- Not an auto-fixer — produce the report; only edit code if the user explicitly asks you to apply a specific finding afterward.
