# sf-git-merge-driver

Git merge driver for Salesforce metadata XML. Auto-resolves conflicts in
profiles, permission sets, 50+ metadata types. TypeScript, oclif sf plugin.

## Commands

All npm scripts delegate to **wireit** — real command + deps live in the
`wireit{}` block of `package.json`, not the `scripts{}` line.

```bash
npm run build        # tsc compile + lint (NOT the binary — see build:bin)
npm run build:bin    # esbuild-bundle lib/ → bin/merge-driver.cjs (what git runs)
npm test             # build + lint + unit + integration + functional + nut
npm run test:unit    # vitest, with coverage
npm run test:integration  # vitest, SERIAL (--maxWorkers=1) — do not parallelize
npm run test:functional   # vitest test/functional
npm run test:nut     # mocha + ts-node on **/*.nut.ts (different runner)
npm run test:mutation     # stryker
npm run lint         # biome check (also lint:fix)
```

Node ≥ 20.

## Architecture

Full design in `DESIGN.md`. Key facts:

- **Two runtimes.** `sf git merge driver install/uninstall` (oclif, one-off)
  writes `.git/config` so git invokes **`bin/merge-driver.cjs`** directly per
  file — a standalone esbuild bundle with no oclif / `@salesforce/core` (~37 ms
  cold start). `sf git merge driver run` is **deprecated** (slow, compat only).
- **Editing `src/` is not enough for git-path behavior.** Rebuild the binary
  (`npm run build:bin`, which depends on `compile`) — git runs the bundled
  `.cjs`, not your source or `lib/`.
- Merge core: `MergeDriver.mergeFiles` parses ancestor/ours/theirs in parallel →
  in-memory merge → `XmlStreamWriter` → atomic rename over `ours`.
- Hexagonal: `src/adapter/` (xml parse/serialize ports), `src/merger/`
  (strategies + composite merge nodes), `src/driver/`, `src/service/`.

## Where changes usually go

- **New metadata-type support / key extractors** → `src/service/MetadataService.ts`.
  This is the highest-churn file (most recent releases add key mappings here).
  Add a matching case to `test/unit/service/MetadataService.test.ts` +
  fixture-based check in `test/integration/XmlMerger.test.ts`.
- Array-merge strategies → `src/merger/nodes/` (keyed/ordered/text nodes).
- XML edge cases → add a numbered fixture under `test/fixtures/xml/NN-*`.

## Gotchas

- `__VERSION__` / `__BUNDLED__` are esbuild `--define` injections (ambient in
  `src/types/globals.d.ts`); undefined in dev/test — every call site guards with
  `typeof`.
- Logging is pure-Node NDJSON (`src/utils/LoggingService.ts`), not SF core
  Logger. `@log()` decorator installs no wrapper unless `SF_LOG_LEVEL=trace`.
- Equality is custom iterative `jsonEqual.ts` (stack-safe), not `fast-equals`.

## Conventions

- Conventional Commits enforced (commitlint + husky). Releases via release-please.
- Format/lint: biome. Pre-commit hooks run lint:staged — don't bypass with
  `--no-verify`.
