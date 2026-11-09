# @sudobility/genuivo_lib

Business logic library for the Genuivo application. It wraps the API hooks from
`@sudobility/genuivo_client` with Zustand state and shared business logic,
exposing a small set of ready-to-use React hooks (and pure helpers) for the UI
layers (`genuivo_app`, `genuivo_app_rn`).

- **Package**: `@sudobility/genuivo_lib` (scoped, published to GitHub Packages, restricted)
- **License**: BUSL-1.1
- **Module format**: ESM only (`"type": "module"`)

## Installation

This package is published to GitHub Packages under the `@sudobility` scope, so
consumers need an `.npmrc` that routes the scope to the GitHub registry (auth
token in the user `~/.npmrc`):

```
@sudobility:registry=https://npm.pkg.github.com
```

Then:

```bash
bun add @sudobility/genuivo_lib
```

Use **Bun** for dependency management (not npm/yarn/pnpm).

### Peer / runtime dependencies

Declared peer dependencies:

```bash
bun add react @tanstack/react-query zustand @sudobility/types @sudobility/genui_types
```

The hooks also import `@sudobility/genuivo_client` and `@sudobility/genuivo_types`
at runtime, so those must be resolvable in the consuming app as well.

## Usage

### useHistoriesManager

Primary hook: combines `genuivo_client` query/mutation hooks + the Zustand cache
store + business logic into one interface.

```ts
import { useHistoriesManager } from "@sudobility/genuivo_lib";

function Dashboard() {
  const {
    histories,      // History[] — fresh server data, falling back to cache
    total,          // global total across all users
    percentage,     // (userSum / globalTotal) * 100
    isLoading,
    error,          // first non-null error (histories > total > mutations)
    isCached,       // true when showing cached data before server responds
    cachedAt,       // ms timestamp of cached data, or null
    createHistory,  // (data) => Promise<void>
    updateHistory,  // (id, data) => Promise<void>
    deleteHistory,  // (id) => Promise<void>
    refresh,        // force a refetch
  } = useHistoriesManager({
    userId: "firebase-uid",   // Optional<string>
    token: firebaseIdToken,   // Optional<FirebaseIdToken>
    autoFetch: true,          // default; set false for manual refresh()
  });
}
```

Note: the config is `{ userId, token, autoFetch? }` only — there is no
`baseUrl`/`networkClient` argument; the underlying client is configured
separately in the consuming app.

### useChatManager

Drives a multi-step, server-rendered chat flow. It holds the current
`IRenderable`, accumulates user input across rounds, and submits to the chat
endpoint via `@sudobility/genuivo_client`.

```ts
import { useChatManager, hasInputControls } from "@sudobility/genuivo_lib";

const {
  currentRenderable, // IRenderable to render (starts with a welcome prompt)
  isLoading,
  error,
  handleAction,      // (value, renderable) => void — record a field's value
  handleSubmit,      // () => Promise<void> — send the accumulated query
  restart,           // () => void — reset to the initial renderable
} = useChatManager({ userId, token });

// hasInputControls(renderable): true if the renderable (or any descendant)
// contains an input/toggle/slider/select layout.
```

### useHistoriesStore

Zustand store providing a per-user, in-memory history cache keyed by user ID.

```ts
import { useHistoriesStore, DEFAULT_CACHE_EXPIRATION_MS } from "@sudobility/genuivo_lib";

// In a component
const histories = useHistoriesStore((s) => s.getHistories("user-1"));

// Outside React
useHistoriesStore.getState().setHistories("user-1", freshData);
```

Actions: `setHistories`, `getHistories(userId, maxAge?)`, `getCacheEntry`,
`addHistory`, `updateHistory`, `removeHistory`, `purgeExpired(maxAge?)`,
`clearAll`. Cache entries older than `maxAge` (default
`DEFAULT_CACHE_EXPIRATION_MS` = 10 minutes) are treated as expired.

### Pure helpers

```ts
import { calculateSum, calculatePercentage } from "@sudobility/genuivo_lib";

calculateSum(histories);                 // sum of all history.value
calculatePercentage(histories, total);   // (userSum / total) * 100; 0 when total <= 0
```

## Project structure

```
src/
├── index.ts                     # re-exports ./business
└── business/
    ├── index.ts                 # re-exports stores + hooks + utils
    ├── stores/historiesStore.ts # per-user Zustand cache
    ├── hooks/
    │   ├── useHistoriesManager.ts
    │   └── useChatManager.ts
    └── utils/calculations.ts    # calculateSum, calculatePercentage
```

## Development

```bash
bun install            # install dependencies
bun run build          # tsc (ESM) + scripts/fix-esm-imports.mjs
bun run clean          # remove dist/
bun run typecheck      # tsc --noEmit
bun run lint           # ESLint (bun run lint:fix to autofix)
bun run format         # Prettier write (format:check to verify)
bun run test:unit      # Vitest run (one-shot)
bun run test:watch     # Vitest watch
bun run verify         # typecheck + lint + test:unit + build (run before commit)
```

Tests use **Vitest** with a jsdom environment. Run them with `bun run test:unit`
(or `bun run test:watch`) — not `bun test`, which invokes Bun's unrelated native
test runner.

The build emits extensionless ESM from `tsc`; `scripts/fix-esm-imports.mjs`
rewrites every relative specifier in `dist/` to a fully-qualified path so the
output resolves under Node's native ESM loader.

## Architecture

```
genuivo_app / genuivo_app_rn   (UI)
        ↓ uses
@sudobility/genuivo_lib        (this package: hooks + Zustand + business logic)
        ↓ uses
@sudobility/genuivo_client     (API hooks, TanStack Query)
        ↓ uses
@sudobility/genuivo_types      (shared type definitions)
```

## Related packages

- **genuivo_types** — shared type definitions
- **genuivo_client** — API client SDK; this library wraps its hooks
- **genuivo_app** — web frontend consuming these hooks
- **genuivo_app_rn** — React Native app consuming these hooks
- **genuivo_api** — backend server (reached indirectly via genuivo_client)
