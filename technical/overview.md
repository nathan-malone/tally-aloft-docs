Technical · Overview · October 2026

# Tally Aloft — Technical Overview

How Tally Aloft is built, and how the design keeps the promises in the [product overview](../product/overview.md). The [docs index](../README.md) lists the document for each part of the system and when to read it.

## System at a glance

```
  Marketing site (Astro, static, tallyaloft.com)

   Web app (React)              Mobile app (Expo: iOS + Android)
          |                                  |
          +---- typed client, generated -----+
          |     from the API's schema        |
          v                                  v
  +--------------- AWS ECS Fargate (one image) ----------------+
  |  API (FastAPI)  |  Taskiq workers  |  Taskiq scheduler (1) |
  +------------------------------------------------------------+
         |                |                |              |
    PostgreSQL          Valkey             S3            KMS
   (the record,      (job broker,    (page photos,   (signing keys)
    audit trail)     rate limits)        PDFs)

  The apps also connect directly to S3 (uploads and downloads through
  short-lived presigned URLs), WorkOS (sign-in) and RevenueCat (purchases).
  The backend calls the Expo Push Service, SES (email) and a vision AI model.
```

- **Backend.** One async Python API serves all three apps. Background workers and a single scheduler run from the same image. The architecture is hexagonal: business rules sit in a pure domain layer, and every external service sits behind a port. Import Linter enforces the boundaries in CI.
- **Clients.** A pnpm and Turborepo monorepo holds a React web app, an Expo React Native app and an Astro marketing site. The two apps share the generated API client, formatting helpers and design tokens, but not screens, and both follow Feature-Sliced Design, which Steiger checks in CI. The site shares only the design tokens.

## Domains

| Host | Serves |
| --- | --- |
| `tallyaloft.com` | Marketing site, legal pages, support and the account-deletion page. `www.tallyaloft.com` redirects here. The support mailbox lives on this domain. |
| `tallyaloft.app` | The web app, plus the iOS and Android app-link files under `/.well-known/`, so email links and notification taps open the apps |
| `api.tallyaloft.app` | The backend API |
| `auth.tallyaloft.app` | WorkOS Authentication API custom domain (the web SDK's `apiHostname`). It must be on the same site as the web app, because it sets the web app's refresh-token cookie. Whether the hosted sign-in pages also get a custom AuthKit domain, such as `login.tallyaloft.app`, is open. |

- **Staging** uses `staging.tallyaloft.app` and `api.staging.tallyaloft.app`, which send `noindex`. WorkOS custom domains are production-only, so staging uses WorkOS's own domains.
- **Email.** Transactional email from SES and WorkOS is sent from `tallyaloft.app`, the domain its links point to. Both domains publish SPF, DKIM and DMARC records, and `tallyaloft.app` receives no mail.
- **DNS** is in Route 53, and certificates come from ACM.

## How the product's promises are kept

| Product promise | How |
| --- | --- |
| Hours are never lost or duplicated | The server is the system of record, and every write goes through the API. Every client write carries an idempotency key, so a retry never creates a second entry. The apps save unsent input and its key on the device until the server confirms the save. |
| Edits are kept, and signed entries can't change | Entry versions, signatures and account events live in append-only tables that the application's database role can't update or delete. Each signature covers one specific entry version (see [Signatures](backend/tech-stack.md#signatures)). |
| Totals are the same everywhere | The server computes every legal total. The apps validate input and format values for display, and never recompute a total. |
| Aircraft prefill, instructor checks, cross-country checks | Reference data from the FAA aircraft and airmen registries and from OurAirports, refreshed on a schedule. |
| PDFs on demand | Rendered with Typst in background workers. Each job is tracked as a row in Postgres, so a crash loses nothing. The app polls the job, or gets a push on mobile, then downloads the file from a short-lived presigned URL. |
| One subscription on every device | Web purchases use Stripe through RevenueCat Billing, and mobile purchases use the App Store and Google Play. RevenueCat combines them into one entitlement per pilot. |
| Old app versions keep working | API changes add fields and endpoints, and remove one only once no supported app release uses it. CI checks each change against every supported release. Below a minimum version, an app shows an "update required" screen. |
| The record survives disasters | Point-in-time recovery, backups copied to a second AWS account and region, and restores rehearsed every quarter. |
| Offline use later | Planned with PowerSync. Version 1 keeps that path open (see [Deferred: offline-first sync](backend/tech-stack.md#deferred-offline-first-sync)). |
