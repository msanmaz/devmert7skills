# Bundle Size

> The commands below use `src/` as a placeholder for your actual audit root — replace it with the real path (`.` for a whole-repo audit, or the specific directory/files you were given) before running them.

The goal is shipping less JavaScript to the browser, because every KB has to be downloaded, parsed, and executed before the page is interactive — this directly feeds into Core Web Vitals (see `references/web-vitals.md`), particularly on slower connections and lower-end devices where parse/execute time dominates over download time.

## Large or duplicate dependencies

Check what's actually in the dependency tree, not just what's imported:

```bash
# npm
npm ls --all 2>/dev/null | grep -iE "moment|lodash$|lodash\." # example: common large/duplicatable libs
npm ls <package-name>   # check for multiple resolved versions of the same package

# pnpm
pnpm why <package-name>

# yarn
yarn why <package-name>
```

Look specifically for:
- **Multiple versions of the same package** resolved in the tree (common after a partial dependency upgrade) — each version ships separately unless deduped.
- **Heavyweight libraries used for a small slice of their functionality** — e.g. all of `moment`/`lodash` imported for one function, when a native alternative or a smaller focused package would do. Only flag this if the library isn't already tree-shaken effectively (see barrel-file note below) and the unused surface is large.
- **Duplicate functionality across libraries** — two date libraries, two state-management libraries, etc., left over from a migration.

If the project has a bundle analyzer configured (`webpack-bundle-analyzer`, `vite-bundle-visualizer`, `rollup-plugin-visualizer`), prefer running it over guessing from `package.json` — it gives you actual shipped bytes per module, not just declared dependencies:

```bash
grep -rn "bundle-analyzer\|bundle-visualizer" package.json vite.config.* webpack.config.* 2>/dev/null
```

If one exists and a build output is available, read its report. If none exists, say so in the finding rather than fabricating byte estimates — "no bundle analyzer configured, byte impact is estimated" is honest; a made-up KB number is not.

## Missing code-splitting / lazy loading

Routes and large, rarely-used components (modals, settings panels, rich editors, chart libraries) that are statically imported instead of lazy-loaded force the user to download them before they're needed, even if they never open that view in the session.

```bash
grep -rln "import.*from ['\"].*routes" src/**/routes.tsx src/**/router.* 2>/dev/null
grep -rn "React.lazy\|lazy(() =>\|import(" src/ --include="*.tsx" | wc -l
```

Compare the route table against which route components are statically imported vs. wrapped in `React.lazy`/dynamic `import()`. A route-level component that's always statically imported is a split candidate unless it's the landing/default route (which needs to load immediately anyway, so splitting it adds a round-trip for no benefit).

## Barrel-file over-importing

A barrel file (`index.ts` that re-exports everything from a directory) can defeat tree-shaking if the bundler can't prove the unused re-exports have no side effects — importing one component from a large barrel can pull in the whole directory's dependency graph.

```bash
grep -rn "export \* from\|export {.*} from" src/**/index.ts 2>/dev/null | wc -l
```

If a barrel file has many re-exports and the project's bundler config doesn't mark the package `"sideEffects": false` in `package.json`, importing even one named export from it can bundle everything behind it. Check whether specific, high-traffic import sites import from the barrel (`import { Button } from '../components'`) vs. the specific file (`import { Button } from '../components/button'`) — the latter is always safe regardless of sideEffects config.

## Dead exports

Exported functions/components/constants that nothing in the codebase imports. These don't cost bytes if tree-shaking works correctly, but they're worth a brief mention if you notice a cluster of them — often a sign of an incomplete removal that's also leaving dead *imported* code behind elsewhere. Don't spend significant audit time hunting for these specifically; they're a minor finding, not a focus area.
