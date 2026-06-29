# mojodoc — Agent guide

## Commands (always via pixi)

```bash
pixi run install          # pnpm install
pixi run build            # pnpm -r build (build all packages first)
pixi run tests            # vitest run
pixi run lint             # eslint packages/*/src/**/*.ts
pixi run format           # prettier --write packages/*/src/**/*.ts
pixi run typecheck        # pnpm -r typecheck
pixi run check            # format:check + lint + test (full pre-PR gate)
pixi run mojodoc ../pkg   # run the CLI on a Mojo package
pixi run docs             # shorthand for `mojodoc . --open`
```

## Architecture

pnpm workspace monorepo (`packages/*`), ESM, strict TypeScript.

**Pipeline:** `mojo doc` JSON → `@mojodoc/parser` → `@mojodoc/transform` → `@mojodoc/renderer` → static HTML

| Package | Role | Entrypoint |
|---------|------|------------|
| `cli` | Commander CLI, file watcher, dev server | `packages/cli/src/index.ts` |
| `parser` | Validate/parse `mojo doc` JSON | `packages/parser/src/types.ts` |
| `transform` | Build DocSite, nav tree, search index, markdown + type cross-refs | `packages/transform/src/transform.ts` |
| `renderer` | HTML templates, CSS (Inferno theme), client JS | `packages/renderer/src/render.ts` |
| `ui` | Placeholder | — |

## Important conventions

- **Build before running** — `pixi run mojodoc` runs the compiled JS; `pixi run build` first or changes won't apply.
- **Tests** live in `tests/*.test.ts` and use `tests/fixtures/sample.json`. Run with `vitest run` (no vitest in watch mode).
- **Config** is auto-detected from `pixi.toml` (name, version) and git remote (source links). No config file needed.
- **Source links** auto-detect from `git remote get-url origin`. Override with `--repository`.
- **No `pnpm-lock.yaml`** — gitignored via `*.lock` pattern.
- **`pixi.lock`** is tracked but binary-marked in `.gitattributes`.
- **ESM only** — `"type": "module"`, all imports use `.js` extensions in source.
- **`verbatimModuleSyntax`** — use `import type` for type-only imports.
- **Live reload** — `--open` starts a dev server with SSE-based reload on `.mojo` changes.

## Data flow details

1. `build()` calls `runMojoDoc()` which shells out to `mojo doc [path]`
2. `parseJson()` validates the JSON and produces typed AST
3. `transform()` scans ALL paths (args, returns, fields, type params, traits) to build a type registry for cross-referencing; also parses `__init__.mojo` for public API extraction
4. `render()` outputs HTML pages, CSS/JS assets, and `search-index.json`

## CI

GitHub Actions on push/PR to `main`, daily at 2am. Matrix: ubuntu-latest + macos-14. Steps: `pixi run install` → `pixi run build` → `pixi run tests`.
