Clients · Structure · October 2026

# Tally Aloft — Client Project Structure

How the web and mobile apps and the marketing site are organized. The repository is a pnpm + Turborepo monorepo with the two apps, the site and a few shared packages. Inside the web and mobile apps, code follows **Feature-Sliced Design (FSD) v2.1**, with import rules checked in CI. The site uses Astro's plain layout (see [The marketing site](#the-marketing-site)). This document defines the parts every version of the apps needs (the monorepo, shared packages, layers, data flow, authentication, configuration and version skew) and describes, with examples, where product screens and features go as they're designed. The libraries named here are in [tech-stack.md](tech-stack.md).

## Repository layout

```
tally-aloft-clients/
├── package.json             # root scripts only; no app code
├── pnpm-workspace.yaml      # apps/*, packages/*, and the version catalog
├── turbo.json               # task graph and caching
├── biome.json               # lint and format for the whole repo
├── .aislop/config.yml       # the AI-slop check for the whole repo
├── .github/workflows/       # CI, web and site deploys, mobile builds
├── docs/
├── apps/
│   ├── web/
│   │   ├── index.html
│   │   ├── vite.config.ts
│   │   ├── components.json  # shadcn: aliases point at @/shared/ui and @/shared/lib
│   │   ├── steiger.config.ts
│   │   ├── e2e/             # Playwright tests
│   │   └── src/             # FSD layers (below)
│   ├── mobile/
│   │   ├── index.ts         # entry: loads expo-router/entry, then configures Unistyles
│   │   ├── app.config.ts
│   │   ├── eas.json
│   │   ├── metro.config.js
│   │   ├── steiger.config.ts
│   │   ├── .maestro/        # Maestro flows
│   │   └── src/             # FSD layers (below)
│   └── site/                # marketing site at tallyaloft.com (Astro)
│       ├── astro.config.ts
│       └── src/
│           ├── pages/       # one file per URL
│           ├── components/
│           └── content/     # Markdown collections: features, principles, legal
└── packages/
    ├── api/                 # @tally/api: generated client, schemas, query options and keys
    ├── format/              # @tally/format: date, time and duration formatting and parsing
    ├── tokens/              # @tally/tokens: design tokens for the apps and the site
    └── tsconfig/            # @tally/tsconfig: shared compiler presets
```

## Monorepo rules

- **Apps depend on packages. Packages never import apps, and apps never import each other.**
- **Packages ship source.** Each package's `exports` points at TypeScript source, and Vite or Metro compiles it as part of the app. Packages use relative imports internally; `compilerOptions.paths` doesn't work across packages compiled this way. The `@/` alias exists only inside apps.
- **Only `exports` is public.** Anything not listed in a package's `exports` stays private, including `@tally/api`'s generated folder. Biome's `noRestrictedImports` also blocks relative paths into `packages/*/src`, which `exports` alone can't stop.
- **Dependencies are declared.** Biome's `noUndeclaredDependencies` fails an import of any package missing from the importing package's `package.json`. pnpm's isolated layout is a second line of defense, as long as Expo works without falling back to `nodeLinker: hoisted`.
- **One copy of shared runtime libraries.** `@tally/api` lists `react` and `@tanstack/react-query` as peer dependencies, and versions come from the pnpm `catalog:`. Two copies of React in a React Native app cause runtime errors.
- **A new package needs a reason two or more apps share.** Code used by one app stays in that app.

`turbo.json` defines these tasks:

| Task | Runs | Notes |
| --- | --- | --- |
| `//#lint` | `biome ci .` | One root task with `cache: false`. Biome is fast, and a root task's cache key wouldn't reliably include files inside `apps/` and `packages/`. |
| `tokens:generate` | Builds `tokens.css` and Unistyles themes | Cached; `dev`, `build`, `typecheck` and `test` depend on it. |
| `transit` | Nothing (an empty task) | `dependsOn: ["^transit"]`. `typecheck` and `test` depend on it, so they re-run when a package they import changes. Without this, a source-only package change can produce a stale cache hit. |
| `typecheck` | `tsc --noEmit` (TypeScript 7); `astro check` (site) |  |
| `steiger` | FSD rules | Web and mobile only. |
| `test` | Vitest (web, packages) or Jest (mobile) |  |
| `build` | `vite build` (web), `astro build` (site) | Mobile builds run on EAS, outside Turborepo. |
| `codegen:check` | Regenerates `@tally/api` and fails on any diff |  |

## Shared packages

### `@tally/api`

The generated API layer, used by both apps.

```
packages/api/
├── openapi.json            # committed snapshot of a tagged backend release
├── openapi-ts.config.ts
└── src/
    ├── gen/                # generated; never edited by hand
    ├── configure.ts        # configureApi({ baseUrl, getAccessToken, clientVersion, platform })
    └── index.ts            # public exports
```

```ts
// packages/api/openapi-ts.config.ts
import { defineConfig } from '@hey-api/openapi-ts'

export default defineConfig({
  input: './openapi.json',
  output: './src/gen',
  plugins: ['@hey-api/client-fetch', 'zod', '@hey-api/sdk', '@tanstack/react-query'],
})
```

- **One client instance.** Hey API generates a single default client, and the generated query and mutation helpers use it. `configureApi` sets that client's base URL and adds request interceptors for the bearer token, the client version and the platform. Each app calls it once at startup, before the first request. Generated helpers use the default client unless passed another one; we never pass one, so there's exactly one configured client.
- **Query keys come from here only.** Apps use the generated `*Options()`, `*InfiniteOptions()` and `*QueryKey()` helpers and never write their own key arrays.
- **Names follow the backend.** Backend operation IDs are `<domain>_<verb>`, so helpers are named like `logbookListFlightsOptions`.
- **Updating.** `pnpm --filter @tally/api sync` downloads `openapi.json` from a **tagged backend release**, never an unreleased branch, and regenerates. The diff in `openapi.json` shows exactly what changed, and TypeScript flags every call site that needs updating. New endpoints never need hand-written client code.

### `@tally/format`

Pure TypeScript helpers for showing dates, times and durations the same way in both apps. No React, no network. Examples of what belongs here:

- Converting the API's ISO date and time strings to and from Temporal types.
- Formatting and parsing durations: decimal hours (`1.3`) and hours and minutes (`1:18`).
- Showing a UTC time in a chosen time zone.

It formats; it doesn't decide. Totals and any other calculated values come from the API. A helper goes here only once both apps need it. `temporal-polyfill` is a peer dependency, versioned through the `catalog:`, so each app loads one copy.

### `@tally/tokens`

The single source for color, spacing, radius and type scale, written in TypeScript as primitives (`blue-600`, `space-4`) and semantic roles (`surface`, `text-muted`, `status-expired`). The `tokens:generate` task writes a Tailwind `@theme` stylesheet for web and the site, and Unistyles themes for mobile, so a color changes in one place.

### `@tally/tsconfig`

Shared presets: `base` (strict, `noUncheckedIndexedAccess`, `verbatimModuleSyntax`, `moduleResolution: "bundler"` so `exports` is honored), `web` (DOM types) and `react-native`. No preset sets `baseUrl`.

## Inside the web and mobile apps: Feature-Sliced Design

Both apps use the same layers under `src/`. A module may import only from layers **below** it, and slices on the same layer can't import each other. The shared packages sit below `shared`, so any layer may import them.

| Layer | Holds | Sliced? |
| --- | --- | --- |
| `app` | Routes, providers, the QueryClient, global styles | No |
| `pages` | One slice per screen | Yes |
| `widgets` | Large UI blocks reused across pages, only when needed | Yes |
| `features` | A complete user action used by more than one page | Yes |
| `entities` | A business object used by more than one feature or page | Yes |
| `shared` | No business logic: UI kit, API setup, auth session, drafts, config, utilities | No; split into segments |

`app` and `shared` exist from day one, along with two pages every version of the apps needs: `pages/sign-in` and `pages/update-required`. Everything else appears as product screens are built.

### How slices appear

FSD v2.1 recommends starting with only `app`, `pages` and `shared`, and keeping code in its page until it's needed elsewhere. **Team rule:** move code down into `entities` or `features` when a second page needs it and it has a clear boundary of its own. Some duplication between pages is cheaper than a wrong abstraction. Don't create empty layers or slices in advance; Steiger flags slices with one or no consumers.

When code moves down, ask these questions in order:

1. **Does it change server data, or run a flow the user starts** (a form, a mutation, a confirm dialog)? It's a **feature**: one user action, named by its verb. Logging a flight is the clearest example (`features/log-flight`), once more than one screen offers it. A feature may import entities.
2. **Does it only read or show one kind of business object** (its view model, `select` mappers, display components)? It's an **entity**, named by its noun. An entity never calls a mutation; actions reach it through props or slots from the code that uses it. A flight row with a delete button is `entities/flight`'s row, with the delete action passed in by the page or feature.
3. **Is it a self-contained block that combines entities and features, used unchanged on two or more pages?** It's a **widget**.
4. **Otherwise** it stays in its page. A screen listing flights would be `pages/logbook`.

An entity is a resource the API returns, which is usually smaller than a backend domain: the backend's `logbook` domain returns flights, so the entity is `entities/flight`. A generated helper's prefix (`logbook…`) names the backend domain that owns the endpoint, not the slice that uses it.

Slice names are kebab-case and match what users would call the thing.

### Slices and segments

Each slice is a folder with an `index.ts` public API and segments grouped by purpose (`ui`, `api`, `model`, `lib`, `config`). Using the examples above:

```
src/
├── entities/
│   └── flight/
│       ├── index.ts           # public API: list each export
│       ├── model/
│       │   └── flight.ts      # view model and select() mappers over generated types
│       └── ui/
│           └── flight-row.tsx
├── features/
│   └── log-flight/
│       ├── index.ts
│       ├── model/
│       │   ├── schema.ts          # form schema, extending the generated Zod schema
│       │   └── use-log-flight.ts  # generated mutation + meta.invalidates
│       └── ui/
│           └── log-flight-form.tsx
└── pages/
    └── logbook/
        ├── index.ts
        └── ui/
            └── logbook-page.tsx   # composes entities and features
```

Rules inside slices:

- Import another slice only through its `index.ts`; never reach into its folders.
- No `export *` in public APIs.
- Relative imports inside a slice, `@/` alias imports between slices.
- `@x` cross-imports between entities are a last resort, only on the entities layer.

## Data flow

This is where each rule of the [client contract](../client-contract.md) lives in the apps.

- **Reading.** Pages and entities call the generated `*Options()` helpers with `useQuery`, adding a `select` mapper from the entity's `model/` when the screen needs a view model. Lists use `*InfiniteOptions()` with `useInfiniteQuery`.
- **Open enums.** Entities map enum values to labels with a fallback for values the build doesn't know, so a new server value never breaks an old build.
- **Writing and invalidation.** A mutation lists the query keys it affects in `meta.invalidates`, using the generated `*QueryKey()` functions. `meta.invalidates` is our convention, typed through TanStack Query's `Register` interface. A `MutationCache.onSuccess` handler in the QueryClient invalidates the listed keys. A mutation can therefore refresh data owned by other slices without importing them: logging a flight can refresh the flight list and the totals shown elsewhere.
- **Drafts and idempotency.** `shared/lib/drafts` creates an idempotency key when a form opens and saves it with the form's values as the pilot types. The mutation sends the same key on every attempt. Both are cleared only when the server confirms success. A `422` `idempotency_key_reused` after an unconfirmed attempt means an earlier attempt saved; the client refetches and tells the pilot the entry is already saved.
- **Version conflicts.** An edit form sends the `base_version` it loaded. On `409` `version_conflict`, it shows the newer version and lets the pilot reapply their change.
- **QueryClient.** `shared/api/query-client.ts` exports a `createQueryClient()` factory with the mutation-cache handler and defaults. The `app` layer creates the single instance and provides it.
- **Update required.** The QueryClient's `QueryCache` and `MutationCache` `onError` handlers watch for the `client_version_unsupported` error code and set a flag in `shared/api`. When it's set, `app` shows `pages/update-required` in place of every route.

## Authentication and session

- **`shared/auth`** holds the session state and a `getAccessToken` function that `configureApi` uses.
- Web: AuthKit's provider lives in `app`, which registers AuthKit's `getAccessToken` with `shared/auth` at startup. `app` reads `useAuth()`, waits until it has loaded, passes `auth` into the router's context, and calls `router.invalidate()` when it changes, so route guards re-run. AuthKit's redirect URI is a route file in `app`.
- Mobile: `shared/auth` owns the PKCE sign-in call, secure-store reads and writes, and token refresh.
- **Sign-in is a page** (`pages/sign-in`), not a feature.
- **Route guards.** Web: TanStack Router `beforeLoad` checks `context.auth`. Mobile: Expo Router `Stack.Protected` with the session state.

## Routing

Route files are thin on both platforms. A route declares its path and inputs, then renders a page from `pages/`. Every page has exactly one route file per app that shows it.

**Web.** TanStack Router file routes live in `src/app/routes/`, and the generated route tree in `src/app/routeTree.gen.ts`. Both are set in the router plugin (`routesDirectory`, `generatedRouteTree`). Every file in the routes folder becomes a route unless its name starts with `-`. The root route uses `createRootRouteWithContext<{ queryClient: QueryClient; auth: AuthState }>()({ … })`. Test files in the routes folder are excluded with the plugin's `routeFileIgnorePattern: '\\.test\\.'`. A list route keeps sort and filters in the URL and prefetches the first page. The page loads further pages with `useInfiniteQuery`, so the cursor never goes in the URL:

```ts
// apps/web/src/app/routes/logbook.tsx
export const Route = createFileRoute('/logbook')({
  validateSearch: logbookSearchSchema,                // ?sort=…&aircraft=…
  loaderDeps: ({ search }) => search,
  loader: ({ context, deps }) =>
    context.queryClient.ensureInfiniteQueryData(logbookListFlightsInfiniteOptions({ query: deps })),
  component: LogbookPage,
})
```

**Mobile.** Expo Router reads routes from `src/app/`, which is also the FSD `app` layer.

- When `src/app/` exists, Expo Router uses it, and every file inside becomes a route. So `src/app/` holds only route files and `_layout.tsx` files, where providers are composed.
- The code those layouts use (the QueryClient factory, Sentry setup, the auth session) lives in `shared/`.
- Unistyles must be configured in a custom entry file rather than in a layout. `apps/mobile/index.ts` imports `expo-router/entry` and then `shared/config/unistyles`, the order Unistyles' Expo Router guide uses, and `package.json` sets `"main": "index.ts"`.

## Configuration

`shared/config/env.ts` validates configuration with Zod at startup and exports a typed object.

- **Web** parses `import.meta.env` (`VITE_*` variables).
- **Mobile** builds the object explicitly: `{ apiUrl: process.env.EXPO_PUBLIC_API_URL, … }`. Expo only inlines `process.env.EXPO_PUBLIC_*` accessed by name, so `z.parse(process.env)` would see nothing. An update bakes in the values present when it's bundled, so CI runs `eas env:pull` for the target environment before `eas update`.

## Version skew

Mobile users run builds that are months old. Beyond the backend's contract rules, the apps add two safeguards:

- `@tally/api` is synced only from tagged backend releases.
- EAS `runtimeVersion` uses the fingerprint policy, so an over-the-air update only reaches binaries whose native code matches.

## UI kits and visual consistency

Each app has its own `shared/ui`; they share tokens and a vocabulary, not code.

- **Web `shared/ui`:** shadcn/ui components on Base UI, styled with Tailwind. `components.json` points shadcn at `@/shared/ui` and `@/shared/lib`, so generated components land inside the layers.
- **Mobile `shared/ui`:** components built with Unistyles, using Expo UI where a platform control is expected.
- **One list of controls.** Both apps implement the same named controls with matching props where the platforms allow. The list lives in `docs/` and is updated when a control is added. Generic controls are added when the first form needs them; a `DurationInput` for flight times is a likely early one.
- **Composition, not atomic folders.** FSD decides where code goes, so `shared/ui` isn't split into atoms and molecules.
  - Compose small primitives into controls, such as a generic `Combobox`.
  - Name components by what they do (`StatusBadge`, not `Atom/Badge`).
  - Build on tokens.
- **No product data in `shared/ui`.** A control filled from an API list is the generic `Combobox` wrapped in the owning entity's `ui` segment.

## The marketing site

`apps/site` is a static Astro site with a plain layout: `pages/` maps one file to each URL, `components/` holds its own components, and `content/` holds Markdown collections. It needs no FSD layers or Steiger, and imports only `@tally/tokens`. Which pages it must have is in the [client tech stack](tech-stack.md#content-rules).

## Where things go

| Adding… | Goes in |
| --- | --- |
| A new backend endpoint | Nothing by hand: run `sync` in `@tally/api` |
| A marketing or legal page | `apps/site/src/pages/`, with its text in `apps/site/src/content/` |
| A screen | `pages/<page>/`, plus a route file in `app` |
| A form or action used on one screen | That page's slice |
| A form or action used on two or more screens | `features/<action>/` |
| A view model or `select` mapper used by two or more screens | `entities/<entity>/model/` |
| A mutation and the queries it invalidates | Where the action lives (page or feature), using `meta.invalidates` |
| A provider | `app/` (web), or `src/app/_layout.tsx` with its code in `shared/` (mobile) |
| Auth session or token handling | `shared/auth/` |
| Draft persistence and idempotency keys | `shared/lib/drafts/` |
| A generic control | `shared/ui/` in each app, and the controls list in `docs/` |
| Environment variables | `shared/config/env.ts` |
| A formatting or time helper both apps need | `@tally/format` |
| A color or spacing value | `@tally/tokens` |

## Enforced boundaries

| Rule | Enforced by |
| --- | --- |
| Layer order, no same-layer slice imports, public API only | Steiger (official FSD linter) in each app |
| No deep or relative imports into packages | Package `exports` + Biome `noRestrictedImports` |
| No undeclared dependencies | Biome `noUndeclaredDependencies`, with pnpm's isolated layout as backup |
| Generated API code is current | `codegen:check` in CI |
| Lint and format | Biome |
| Patterns AI coding agents leave behind | aislop (`make slop.check`) |

```ts
// apps/web/steiger.config.ts (mobile is the same)
import { defineConfig } from 'steiger'
import fsd from '@feature-sliced/steiger-plugin'

export default defineConfig([
  ...fsd.configs.recommended,
  {
    rules: {
      // Split forbidden-imports into its two halves for clearer errors.
      'fsd/forbidden-imports': 'off',
      'fsd/no-cross-imports': 'error',
      'fsd/no-higher-level-imports': 'error',
      'fsd/no-wildcard-exports': 'error',
      'fsd/import-locality': 'error',
    },
  },
])
```

Steiger is in beta. If it blocks us, dependency-cruiser can express the same layer rules.

## Tests

| What | Where | Runner |
| --- | --- | --- |
| Shared packages | Next to the code, `*.test.ts` | Vitest |
| Web components and hooks | Next to the code, `*.test.tsx`; route files' tests are excluded from routing by `routeFileIgnorePattern` | Vitest + React Testing Library |
| Mobile components and hooks | Next to the code, except route files: their tests go in `src/__tests__/`, because any file in `src/app/` becomes a route | Jest (`jest-expo`) + React Native Testing Library |
| Web end-to-end | `apps/web/e2e/` | Playwright |
| Mobile end-to-end | `apps/mobile/.maestro/` | Maestro |

The drafts module has a required test: an attempt whose response is lost, followed by a resubmit, sends the same idempotency key.

## CI

On every pull request: `turbo run //#lint typecheck steiger test codegen:check`, the AI-slop check (`aislop ci`), then Playwright against a production build of the web app. When the repository is set up, its `Makefile` will wrap these tasks, and CI will call one make target per check and an end-to-end target, so the README, `CLAUDE.md` and CI share one copy of each command (see [make commands](../make-commands.md)). Merges to `main` deploy the web app and the site to their S3 buckets and CloudFront distributions, and run EAS builds for mobile. Store releases and over-the-air updates are triggered by tags; the update job pulls EAS environment variables, publishes the update and uploads its source maps to Sentry.
