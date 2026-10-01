# Core Web Vitals (Live Measurement)

This category requires an actual running page — it's not something you can determine from reading source code alone, so it's conditional: run it when a dev server (or any reachable URL for the app) is available, skip it cleanly otherwise. Say explicitly in the report which happened; don't silently omit the section.

## Prerequisites

You need the chrome-devtools MCP tools. Load them before use:

```
MCPSearch({ query: "select:mcp__chrome-devtools__performance_start_trace,mcp__chrome-devtools__performance_stop_trace,mcp__chrome-devtools__performance_analyze_insight,mcp__chrome-devtools__lighthouse_audit,mcp__chrome-devtools__navigate_page,mcp__chrome-devtools__new_page" })
```

If these tools aren't available in this environment at all (no chrome-devtools MCP configured), say so and skip the category — don't fall back to guessing vitals from static analysis, since they're meaningless without real paint/input timing data.

**Finding the URL:** if the user didn't give you one, check for a running dev server the way you would for any local verification task (common ports, a `dev-watch status`-style tool if the project has one, or ask). Don't start a dev server yourself as a side effect of an audit — that's a bigger action than "run a read-only check," confirm with the user first if nothing is already running.

## What to measure

The three Core Web Vitals and their rough thresholds (good / needs improvement / poor) — use these to judge severity, not as hard pass/fail gates, since real-user-condition thresholds differ from a single local trace:

| Metric | Good | Needs Improvement | Poor | What it measures |
|---|---|---|---|---|
| **LCP** (Largest Contentful Paint) | ≤ 2.5s | 2.5–4s | > 4s | Time until the largest visible element renders — usually a hero image, a large text block, or the main content container |
| **CLS** (Cumulative Layout Shift) | ≤ 0.1 | 0.1–0.25 | > 0.25 | How much visible content shifts position unexpectedly during load/interaction |
| **INP** (Interaction to Next Paint) | ≤ 200ms | 200–500ms | > 500ms | Responsiveness of the page to user interaction (click, tap, key press) across the session, not just one event |

## How to run it

1. Use `lighthouse_audit` against the target URL for a quick overall read — it reports all three vitals plus other diagnostics (render-blocking resources, unused JS/CSS, image sizing) in one pass. Start here; it's cheaper than a manual trace and usually points you at what's worth digging into further.
2. For a specific suspected interaction (a slow click, a janky scroll, a component you already flagged in the re-renders category), use `performance_start_trace`, perform the interaction via `navigate_page`/the page tools, then `performance_stop_trace`, and run `performance_analyze_insight` on the result to get the actual call stack / timing breakdown rather than guessing from the Lighthouse summary alone.
3. Always navigate to the actual state you care about before tracing — a trace of the empty loading state tells you nothing about a vitals problem that only shows up once data is populated. If the page has meaningfully different states (logged out, logged in, populated, empty), be explicit in the report about which state you measured.

## Connecting a vitals finding back to source

A Lighthouse/trace result tells you *that* something is slow, not always *why* in terms of a specific component. Use the trace's flagged resources (a specific chunk, a specific long task) and cross-reference:
- A long task during initial render → look for synchronous expensive work in a top-level component (see `references/general-smells.md`).
- A layout shift → look for images/embeds without explicit width/height, or content injected above existing content after load (e.g. an ad slot, a banner that loads late).
- Poor LCP with a late largest element → check whether that element's data fetch is blocked behind other work that could be parallelized (see `references/network-data-fetching.md`) or whether the image itself is unoptimized (see `references/general-smells.md`).

Report the actual measured number in the finding ("LCP measured at 4.2s on the populated dashboard state") rather than a vague "this seems slow" — that's the whole value of doing live measurement instead of just reading code.
