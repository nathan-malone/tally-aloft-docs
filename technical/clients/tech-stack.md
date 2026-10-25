Clients · Tech stack · October 2026

# Tally Aloft — Client Tech Stack

Tally Aloft has two apps, a web app and a mobile app for iOS and Android, both written in React, and a static marketing site. All three live in one pnpm + Turborepo monorepo. The backend lives in its own repository ([backend tech stack](../backend/tech-stack.md)); the two connect through the backend's OpenAPI schema and the [client contract](../client-contract.md). This document records what the clients are built with and why. How the code is organized is in [structure.md](structure.md).

## Principles

- **Share logic, not screens.** Both apps share the API client, validation, formatting and design tokens. Each app has its own components, because desktop logbook entry and phone logging are different jobs.
- **The API is the contract.** Request and response types, Zod schemas and query helpers are generated from the backend's OpenAPI schema. Nobody hand-writes an API type.
- **The server owns the rules.** Totals, currency, entitlements and signature validity come from the backend. Clients validate input shape and format values for display; they never recompute a legal total. Version 1 requires a connection.
- **Never lose what the pilot typed.** Unsent form input survives a dropped connection or a killed app (see Cross-cutting client behavior).
- **Fast tools.** The toolchain is Rust- and Go-based where possible (Vite 8 with Rolldown, Biome, TypeScript 7), so lint, type-check and build stay fast as the codebase grows.

## Shared foundation

| Concern | Choice | Notes |
| --- | --- | --- |
| Language | TypeScript 7 | Native compiler, about 10x faster type-checks. 7.0 has no stable programmatic API yet. Type-checking uses TS 7; `@tally/api` and `apps/site` declare `"typescript": "npm:@typescript/typescript6@^6"`, so Hey API's generator and `astro check` get the TS 6 API. `baseUrl` is not used in any tsconfig. |
| UI library | React 19.3 | React Compiler enabled in both apps for automatic memoization. |
| Package manager | pnpm workspaces | Shared dependency versions pinned once with a pnpm `catalog:` so the apps and the site use the same React, and both apps the same TanStack Query. |
| Task runner | Turborepo | Caches `typecheck`, `test` and `build` per package. |
| Lint and format | Biome 2 | One root `biome.json`, run as a single root task. Biome's `.astro` support is experimental. We leave `html.experimentalFullSupportEnabled` off, so Biome checks the frontmatter but not the template markup, and an override turns off the four rules Biome's docs list for that mode (`useConst`, `useImportType`, `noUnusedVariables`, `noUnusedImports`). |
| API client | Hey API `openapi-ts` | Generates a typed fetch client, Zod schemas and TanStack Query options and keys. Responses aren't validated at runtime: the backend's schema already fixes their shape, and strict validation would break old builds on an additive change. Still 0.x, so its version is pinned exactly. |
| Server state | TanStack Query 5 | Fetching, caching and invalidation in both apps. Chosen over RTK Query for its direct integration with TanStack Router loaders and its official PowerSync integration for future offline support. |
| Forms | React Hook Form + `@hookform/resolvers` | Zod resolver; mobile inputs use `Controller`. |
| Validation | Zod 4 | Generated API schemas plus form schemas built from them. |
| Dates and times | Temporal via `temporal-polyfill` | `PlainDate` for flight dates, `Instant` and `ZonedDateTime` for UTC times, `Duration` for block times. The API sends ISO strings; `@tally/format` converts them to and from Temporal types. The polyfill (about 20 kB gzipped) stays until Safari and Hermes ship Temporal. |
| Error monitoring | Sentry | Web errors with source maps; native iOS and Android crash reports. `sendDefaultPii: false`, and request bodies are never attached. Tracing to the backend follows the client contract; confirm the React Native SDK supports `propagateTraceparent` when setting it up. SigNoz was evaluated and lacks source-mapped error tracking, session replay and native crash reporting. |
| CI | GitHub Actions | Same platform as the backend. |

## Web app

| Concern | Choice | Notes |
| --- | --- | --- |
| Build | Vite 8 | Rolldown bundler. `@vitejs/plugin-react` v6 no longer runs Babel, so React Compiler runs through `@rolldown/plugin-babel` with `babel-plugin-react-compiler`: `plugins: [react(), babel({ presets: [reactCompilerPreset()] })]`. |
| Routing | TanStack Router | File-based routes, typed path params, search params validated with Zod, loaders that prefetch with TanStack Query, `beforeLoad` route guards. |
| Styling | Tailwind CSS 4 | Theme generated from `@tally/tokens`. |
| Components | shadcn/ui on Base UI | Base UI has been shadcn's default since July 2026. Component source is copied into `shared/ui` and owned by us. |
| Icons | `lucide-react` |  |
| Tables | TanStack Table + TanStack Virtual | Headless logbook grid with column visibility and virtualized rows. Sorting and filtering are done by the server (see [backend reads](../backend/structure.md#application--use-cases-and-ports)), and rows load page by page with `useInfiniteQuery`. |
| Authentication | `@workos-inc/authkit-react` | `getAccessToken()` supplies the bearer token. In production, the SDK's `apiHostname` is the WorkOS Authentication API custom domain, `auth.tallyaloft.app`, which is on the app's own site. The refresh token therefore lives in a first-party HTTP-only cookie. WorkOS offers custom domains only in production, so local and staging builds turn on `devMode`, which keeps the refresh token in local storage. Production never does. Access tokens are never stored outside the SDK. |
| Subscriptions | `@revenuecat/purchases-js` | RevenueCat Billing with Stripe. Configured with the pilot's `PilotId`, read from the API after sign-in, as the RevenueCat app user ID. |
| Monitoring | `@sentry/react` | Source maps uploaded from CI. |
| Unit and component tests | Vitest + React Testing Library | jsdom environment. |
| End-to-end tests | Playwright |  |
| Hosting | Amazon S3 + CloudFront at `tallyaloft.app` | A CloudFront Function rewrites client routes to `index.html`, except `/.well-known/*`, which serves the iOS and Android app-link files as JSON. `index.html` is `no-cache`; hashed assets are cached as immutable. CloudFront sets a strict Content Security Policy whose `connect-src` lists only the API, S3, WorkOS, RevenueCat and Sentry origins. Deployed from GitHub Actions with OIDC, so no long-lived AWS keys exist. |

## Mobile app

| Concern | Choice | Notes |
| --- | --- | --- |
| Framework | Expo SDK 58 (React Native 0.88) | New Architecture. Development builds, not Expo Go, because of native modules (Unistyles, the document scanner) and the auth URL scheme. React Compiler enabled in the app config. SDK 58 shipped on October 21, 2026, so confirm each native library supports it at setup. |
| Routing | Expo Router | Native tabs, which pick up the iOS 26 Liquid Glass tab bar. `Stack.Protected` guards signed-in screens. |
| Styling | Unistyles 3 | `StyleSheet`-style API; themes built from `@tally/tokens`. Requires the New Architecture and `react-native-nitro-modules`, and is configured in the app's entry file before any screen loads. |
| Native controls | Expo UI | SwiftUI and Jetpack Compose controls where users expect the platform's own: date and time pickers, menus, sheets. |
| Animation and gestures | Reanimated 4 + Gesture Handler | Runs on the UI thread. |
| Lists | FlashList v2 | Flight lists, endorsements, search results. |
| Icons | `lucide-react-native` | Uses `react-native-svg`. |
| Authentication | WorkOS AuthKit, PKCE public client | System browser via `expo-web-browser`, tokens in `expo-secure-store` (Keychain and Keystore), refresh on expiry. This flow is our design, built from WorkOS's Expo guide. |
| Subscriptions | `react-native-purchases` | RevenueCat SDK for App Store and Google Play. `Purchases.logIn()` with the pilot's `PilotId` after sign-in, so web and store purchases merge into one entitlement. |
| Push notifications | `expo-notifications` | Gets an Expo push token for the backend, which sends through the Expo Push Service. Handles the permission prompt (Android 13 and later included), Android channels and foreground display. Android delivery needs the Firebase config file; iOS needs an APNs key uploaded to EAS. |
| Local storage | `expo-sqlite/kv-store` | Unsent form drafts and their idempotency keys. |
| Logbook page capture | `react-native-document-scanner-plugin` | Wraps VisionKit (iOS) and ML Kit (Android) for edge detection and perspective correction. Images upload straight to S3 with a presigned POST. |
| Files and sharing | `expo-file-system`, `expo-sharing` | Download generated PDFs and hand them to the share sheet to save, print or send. |
| Deep links | Universal links (iOS) and App Links (Android) | Email links and notification taps open the right screen. The production app declares `applinks:tallyaloft.app` and an Android intent filter with `autoVerify` for `tallyaloft.app`, whose `/.well-known/` files the web app serves. |
| Monitoring | `@sentry/react-native` | Source maps and debug symbols upload automatically from EAS Build. For EAS Update, CI exports source maps (`npx expo export --dump-sourcemap`), publishes the update, then uploads them with `npx @sentry/expo-upload-sourcemaps dist`. |
| Unit and component tests | Jest via `jest-expo` + React Native Testing Library | What Expo documents and supports. Vitest, used everywhere else, has no official React Native support. Switch when it does, or when a maintained plugin works with the current Expo SDK, native module mocks included. |
| End-to-end tests | Maestro | YAML flows in `apps/mobile/.maestro/`, run on simulators in CI. |
| Build and release | EAS Build, EAS Submit, EAS Update | `runtimeVersion` uses the `fingerprint` policy, so an over-the-air update only reaches binaries with matching native code. |

## Marketing site

The public site at `tallyaloft.com`: features, pricing, principles, support and legal pages. It never calls the API, with one possible exception: a later contact form (see Content rules).

| Concern | Choice | Notes |
| --- | --- | --- |
| Framework | Astro 7 | Builds every page as complete HTML and ships no JavaScript by default. Pages paint fast, preview well when shared, and can be read by search engines without running scripts. A Vite SPA builds pages in the browser, which suits the signed-in app but not public pages. Astro is MIT-licensed. Its team joined Cloudflare in January 2026; static output keeps the site independent of any host. |
| Content | Astro content collections | Features, principles and legal pages are Markdown, with frontmatter validated by Zod. A changelog or blog can be added later as another collection. |
| Styling | Tailwind CSS 4 | The same `@tally/tokens` theme as the web app. |
| Interactivity | `@astrojs/react` islands | Added only when a page needs one, such as a pricing toggle. Components are the site's own: apps never import each other. |
| Type checking | `astro check` | Needs the TS 6 API (see Language). |
| Hosting | Amazon S3 + its own CloudFront distribution | `site` is set to `https://tallyaloft.com` for canonical URLs and the sitemap. Astro builds `/pricing` as `pricing/index.html`, so a viewer-request CloudFront Function maps `/pricing` and `/pricing/` to that file. Unlike the web app's function, which sends every route to `index.html`, it doesn't rewrite unknown paths. The same function redirects the `www` host. CloudFront maps 403 and 404 from S3 to Astro's `404.html` with status 404. A response headers policy sets a strict CSP (`default-src 'self'`, `frame-ancestors 'none'`) and HSTS. Files under `_astro/` are cached as immutable, and HTML is revalidated on every request. Deployed from GitHub Actions with OIDC. |

### Content rules

- **Required pages.**
  - **Both stores:** a privacy policy URL, linked from the store listings and the apps.
  - **App Store:** a support URL, and Terms of Use, because the app sells auto-renewing subscriptions.
  - **Google Play:** a page for deleting an account outside the app, with its URL entered in Play Console. The page:
    - names the app as it appears on the store listing
    - says to cancel any store subscription first
    - lists what is kept and for how long (instructor endorsement records for 3 years; backups for up to 35 days)
    - links to deletion in the web app, and gives the support email as a second way to ask
- **Pricing.** The page shows web prices. Store prices vary by country, so it doesn't promise an exact in-app price.
- **Contact.** Start with a published support email address at `tallyaloft.com`. A contact form, if one is added later, is a plain HTML form that posts to a rate-limited backend endpoint with bot protection, not to a third-party form service. A plain form post needs no CORS change.

## Cross-cutting client behavior

- **Environments.** Development, staging and production each have their own API URL, WorkOS client ID and redirect URIs, RevenueCat keys, Firebase config file and Sentry DSN. Web reads `VITE_*` variables. Mobile reads `EXPO_PUBLIC_*` variables from EAS environments, and `app.config.ts` picks the bundle ID, URL scheme, app-link domain and Firebase file per `APP_VARIANT` (hosts are in the [technical overview](../overview.md#domains)). Client variables are public by definition, so no secrets go in them.
- **Backend contract.** Both apps follow the [client contract](../client-contract.md): version headers and the update-required screen, idempotency keys, version conflicts, open enums, error codes, jobs and tracing. [structure.md](structure.md#data-flow) shows where each lives in the apps.
- **Unsent drafts.** A form's values and its idempotency key are saved on the device as the pilot types: in local storage on web, and in `expo-sqlite/kv-store` on mobile. They are cleared when the server confirms the save. A lost connection or a killed app loses nothing, and a resubmit can't create a duplicate.
- **Entitlements.** Feature gating reads entitlements from our API. After a purchase, the client calls the API's entitlement refresh so access starts at once.
- **Generated PDFs.** They arrive through the contract's job flow, then download as a browser download on web, or through `expo-file-system` and the share sheet on mobile.

## Deferred: offline support

Version 1 is online-only. If offline use is added, the plan is PowerSync, which has web and React Native SDKs and an official TanStack Query integration, so screens keep their query hooks while reads move to a local SQLite database. The server side is in the [backend tech stack](../backend/tech-stack.md#deferred-offline-first-sync).
