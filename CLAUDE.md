# Genuivo Lib

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it.

Business logic library with Zustand stores for the Genuivo application.

**npm**: `@sudobility/genuivo_lib` (restricted, BUSL-1.1)

## Tech Stack

- **Language**: TypeScript (strict mode)
- **Runtime**: Bun
- **Package Manager**: Bun (do not use npm/yarn/pnpm for installing dependencies)
- **Build**: TypeScript compiler (ESM)
- **Test**: Vitest
- **State**: Zustand 5
- **Data Fetching**: TanStack Query 5

## Project Structure

```
src/
├── index.ts                     # re-exports ./business
└── business/
    ├── index.ts                 # re-exports stores + hooks + utils
    ├── stores/
    │   ├── historiesStore.ts    # per-user Zustand cache store
    │   └── historiesStore.test.ts
    ├── hooks/
    │   ├── useHistoriesManager.ts   # unified histories hook (+ .test.ts)
    │   └── useChatManager.ts        # multi-step chat/renderable flow
    └── utils/
        ├── calculations.ts      # calculateSum, calculatePercentage (+ .test.ts)
        └── index.ts
```

## Commands

```bash
bun install            # install deps (Bun only — not npm/yarn/pnpm)
bun run build          # tsc (ESM) + scripts/fix-esm-imports.mjs
bun run clean          # remove dist/
bun run test:unit      # Vitest run (one-shot) — NOT `bun test`
bun run test:watch     # Vitest watch
bun run typecheck      # tsc --noEmit
bun run lint           # ESLint (lint:fix to autofix)
bun run format         # Prettier write (format:check to verify)
bun run verify         # typecheck + lint + test:unit + build
bun run prepublishOnly # clean + build (runs on publish)
```

`bun test` runs Bun's native test runner (there is no `test` script); always use
`bun run test:unit`. `verify` is the full gate (69 tests across 3 files).

## Key Concepts

### useHistoriesStore

Zustand store providing per-user client-side cache with operations: `set`, `get`, `add`, `update`, `remove`. Keyed by user ID for multi-user support.

### useHistoriesManager

Unified hook that combines genuivo_client hooks + Zustand store + business logic:

- **Config**: `{ userId, token, autoFetch? }` only — there is NO
  `baseUrl`/`networkClient` arg (the client is configured in the consuming app).
  The JSDoc examples in `useHistoriesManager.ts` still show a stale
  `baseUrl`/`networkClient` config — the real `UseHistoriesManagerConfig` does not.
- **Percentage calculation**: `(userSum / globalTotal) * 100` (via `utils/calculations`)
- **Cache fallback**: returns cached data when server hasn't responded yet
- **Auto-fetch**: fetches on mount when `autoFetch` is enabled (default)
- **Token reactivity**: resets the fetch guard when the token changes

This is the primary hook consumed by UI layers (genuivo_app, genuivo_app_rn).

### useChatManager

Drives a multi-step, server-rendered chat flow over `@sudobility/genuivo_client`'s
`useChat`. Config is `{ userId, token }`. Holds the current `IRenderable`,
accumulates input across rounds via `useRef`, and exposes `handleAction`,
`handleSubmit`, `restart`. Also exports `hasInputControls(renderable)`.

### utils/calculations

Pure helpers: `calculateSum(histories)` and
`calculatePercentage(histories, total)` (returns `0` when `total <= 0`).

## Dependencies

- Peer deps declared in package.json: `react` (>=18), `@tanstack/react-query`
  (>=5), `zustand` (>=5), `@sudobility/types` (^1.9.78),
  `@sudobility/genui_types` (^0.0.5).
- Hooks also import `@sudobility/genuivo_client` and `@sudobility/genuivo_types`
  at runtime, but these are only in `devDependencies` — consumers must provide
  them. (Note the name mismatch vs. the declared `genui_types` peer dep.)

## Architecture

```
genuivo_app / genuivo_app_rn
    ↓ uses
@sudobility/genuivo_lib (this package)
    ↓ uses
@sudobility/genuivo_client (API hooks)
    ↓ uses
@sudobility/genuivo_types (type definitions)
```

## Related Projects

- **genuivo_types** — Shared type definitions; imported transitively via genuivo_client
- **genuivo_client** — API client SDK; this library wraps its hooks with business logic and Zustand state
- **genuivo_app** — Web frontend that consumes `useHistoriesManager` from this library
- **genuivo_app_rn** — React Native app that consumes `useHistoriesManager` via file: links
- **genuivo_api** — Backend server; this library communicates with it indirectly through genuivo_client

## Coding Patterns

- `useHistoriesManager` is the primary hook -- it orchestrates genuivo_client hooks + Zustand store into a single unified interface for UI layers
- Zustand store (`useHistoriesStore`) is keyed by `userId` for per-user cache isolation
- Percentage calculation: `(userSum / globalTotal) * 100` -- this is the core business metric
- `isCached` flag indicates when the UI is showing stale cached data before the server responds
- `autoFetch` (default: true) triggers data fetching on mount; use `autoFetch: false` for manual control
- Token reactivity: changing the auth token resets the store state to prevent stale cross-user data
- `useRef` is used to prevent duplicate fetch calls on React strict-mode double-mount

## Gotchas

- Zustand store is in-memory only -- there is no persistence; data is lost on page refresh or app restart
- Cache is isolated per `userId` -- switching users shows a fresh state (not another user's data)
- Token change resets the entire store state -- this is intentional to prevent data leakage between users
- `useRef` guards prevent duplicate fetches on mount; be careful not to break this guard when modifying the hook
- This is a published package (`@sudobility/genuivo_lib`, GitHub Packages, ESM-only) -- coordinate breaking changes with genuivo_app and genuivo_app_rn
- Build is a two-step: `tsc` emits extensionless ESM, then `scripts/fix-esm-imports.mjs` rewrites every relative specifier in `dist/` to a fully-qualified `.js` path so Node's native ESM loader resolves them. Don't drop that post-build step.
- `.npmrc` routes `@sudobility` to `https://npm.pkg.github.com`; the auth token lives in the user `~/.npmrc`, not this repo.

## Git Workflow

- Do not use feature branches for code changes. Always stay on the current branch.
