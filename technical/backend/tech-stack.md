Backend · Tech stack · October 2026

# Tally Aloft — Backend Tech Stack

One backend API serves the web, iOS and Android clients. This document records what the backend is built with and why. The rules the API and every client follow are in the [client contract](../client-contract.md). How the code is organized is in [structure.md](structure.md).

## Principles

- **Async end to end.** Every I/O library is asyncio-native. CPU-heavy work (PDF rendering, image preprocessing) runs in background workers, never on the request event loop.
- **The server is the system of record.** A logbook is a legal record. All writes go through the API, which validates them, versions them and protects signed entries.
- **Buy the undifferentiated parts.** Identity, billing across app stores, email delivery and push delivery use managed services. Engineering time goes to the logbook, the audit engine and data integrity.

## Runtime and core

| Concern | Choice | Notes |
| --- | --- | --- |
| Language | Python 3.14 | Move to 3.15 once the dependencies publish 3.15 wheels. |
| Web framework | FastAPI | OpenAPI schema is the contract for all three clients. |
| ASGI server | Uvicorn | FastAPI's default. Fast enough for a database-bound API. |
| Validation | Pydantic v2 | HTTP request and response models, settings, and LLM output schemas. |
| ORM | SQLAlchemy 2 (async) | Declarative table models in the Postgres adapter, mapped to and from domain dataclasses; Core for list queries. Chosen over SQLModel, which is still 0.0.x and whose shared table-and-schema class this design doesn't use. |
| Database driver | asyncpg | Via SQLAlchemy's `postgresql+asyncpg` dialect. |
| Database | PostgreSQL 19 | See the version note below. Native `uuidv7()` (Postgres 18 and later) for primary keys. |
| Migrations | Alembic | Async `env.py`. Every schema change ships as a reviewed migration (see Operations). |
| Configuration | pydantic-settings | Typed settings from environment variables; secrets injected by ECS from AWS Secrets Manager. |
| Cache and broker | Valkey (Redis-compatible) | Task broker, rate-limit counters and short-lived caches. Nothing durable lives only here: job state is in Postgres. |
| Background jobs | Taskiq + `taskiq-redis` `RedisStreamBroker` | Async-native, with acknowledgements: a killed worker's message is redelivered after the stream's `idle_timeout`, which is set above the longest task's run time. Delivery is at least once; the job row's claim makes that safe (see [structure.md](structure.md#background-jobs)). The scheduler runs as exactly one instance (see Infrastructure), because Taskiq fires a periodic task once per running scheduler. Chosen over Arq, which is maintenance-only (0.28.0, April 2026), and over Postgres-backed queues, whose asyncpg-native options are pre-1.0. `taskiq-fastapi` is still alpha, so task dependencies stay plain. |
| Outbound HTTP | aiohttp | One shared `ClientSession` per process for push, the WorkOS key set, reference-data downloads and other outbound calls. Pydantic AI's model providers bring their own HTTP clients. |
| AWS APIs | aiobotocore | Async AWS SDK for S3, SES (sending), SQS (email events) and KMS (signing). |

**PostgreSQL version.** Postgres 19 reaches general availability on November 19, 2026, and RDS offers it only in Preview until it lists 19 as generally available. Production launches on 19 if RDS has it by then, otherwise on 18 with a later RDS Blue/Green upgrade. Nothing may depend on 19-only features, and local development and tests use production's major version.

## Product services

| Concern | Choice | Notes |
| --- | --- | --- |
| Authentication | WorkOS AuthKit | Passkeys, MFA and social sign-in. WorkOS also sends the sign-in, verification and password-reset emails. Free to 1M monthly active users. Custom domains are a paid add-on, listed at $99 a month; confirm whether that's per domain. The web app needs the Authentication API domain, and the Email domain is recommended. The API verifies access tokens locally with PyJWT against the key set at `https://api.workos.com/sso/jwks/{client_id}`, fetched with the shared aiohttp session and cached, so verification never blocks the event loop. It checks an issuer per environment (production's Authentication API custom domain, `auth.tallyaloft.app`, changes `iss`) and a required audience set in the JWT template. If iOS offers Google or another social login, it must also offer a login that meets App Store guideline 4.8; Sign in with Apple is the simplest choice. |
| Object storage | Amazon S3 via aiobotocore | Logbook page photos, generated PDFs and endorsement records. Files never pass through the API. Uploads use presigned POST policies that S3 enforces (`content-length-range`, `Content-Type`), under keys the server chooses and scopes to the pilot. They land in an `incoming/` prefix that a lifecycle rule expires, and move to their final key once the server has checked their contents. Downloads use presigned GET URLs that set `response-content-disposition`. Buckets are versioned, noncurrent versions expire after 35 days, and AWS Backup copies them with the database. |
| Paper-logbook digitization | Pydantic AI + a vision LLM | Each page image returns a validated Pydantic model of draft entries. The provider is configuration, not code, and must contractually neither retain nor train on page images. Page runs count against a per-pilot quota set by the purchased entitlement. Output is always "AI-drafted, pilot-verified": the pilot confirms every entry against the source image. |
| PDF generation | Typst (`typst` Python package) | Printable logbooks and interview packets. The compiler ships in the wheel, and data is passed in as JSON. Runs in Taskiq workers and reaches clients through the job flow (see the [client contract](../client-contract.md)). |
| Subscriptions | RevenueCat + Stripe | RevenueCat unifies App Store, Google Play and Stripe (through RevenueCat Billing) into one entitlement per pilot; the backend has no direct Stripe integration. The RevenueCat app user ID is the pilot's internal ID (`PilotId`) on every platform, so purchases merge and survive a change of identity provider. Webhooks are verified by their configured Authorization header and HMAC signature, deduplicated on event ID, and followed by a `GET /subscribers` call for the current state. After a purchase, clients call an entitlement refresh endpoint that makes the same call, so access doesn't wait for the webhook. |
| Push notifications | Expo Push Service | One HTTP API for iOS and Android, called with aiohttp in batches of up to 100, with an Expo access token (push security enabled). Free. Clients register Expo push tokens; a scheduled job reads push receipts and deletes tokens reported as `DeviceNotRegistered`. The APNs key and FCM credentials live in EAS, not in the backend. Payloads carry no personal data. Chosen over direct FCM HTTP v1, which needs Workload Identity Federation with a custom credentials supplier on Fargate and React Native Firebase on the client. |
| Transactional email | Amazon SES | Reminders, job notices and account notices, sent with aiobotocore from `tallyaloft.app`. Click tracking stays off, because rewritten links stop email links from opening the apps. Bounce and complaint events go SES → SNS → SQS and are read by a worker, so there's no public endpoint; bad addresses are suppressed. |
| Observability | Pydantic Logfire | OpenTelemetry-based traces, logs and errors, with integrations for FastAPI, asyncpg, aiohttp, Redis and Pydantic AI. Pydantic AI is instrumented with `include_content=False` and `include_binary_content=False`, so page images and extracted entries stay out of traces. See Tracing in the [client contract](../client-contract.md). |

## Aviation domain

| Concern | Choice | Notes |
| --- | --- | --- |
| Instructor signatures | Ed25519 keys in AWS KMS; `cryptography` verifies | See [Signatures](#signatures). |
| Audit trail | Append-only Postgres tables | Entry versions, signatures and account events. Migrations run as the owner role; the application's role has only `SELECT` and `INSERT` on these tables. Only the purge role can delete from them (see Data ownership). |
| Airports | OurAirports dataset | Airport identifiers and coordinates for cross-country distance checks. Refreshed on a schedule by a Taskiq job. |
| Aircraft and airmen | FAA Aircraft Registry and Airmen Certification data | Prefill aircraft by tail number and check instructor certificates. Refreshed on a schedule. |

### Signatures

- **Immutable versions.** A signed entry version never changes. A correction creates a new version that needs a new signature, and the earlier version and its signature stay valid.
- **Fresh MFA.** Signing requires an MFA challenge completed in the last 5 minutes and an explicit attestation by the instructor. The API checks the token's `auth_time` and its MFA evidence. A stale token gets `401` with code `reauthentication_required`, and the client restarts sign-in with `max_age`. Confirm which claim carries the MFA evidence against WorkOS's step-up documentation at setup.
- **What is signed.** The signature covers the RFC 8785 canonical JSON (`rfc8785` package) of: the entry version's content and IDs, the student's `PilotId`, the signer's `PilotId`, CFI certificate number and REED, the attestation text version, the timestamp and the canonicalization version. The metadata is therefore tamper-evident too. Numbers in the payload are integers (durations in minutes), and decimals are strings.
- **Keys.** There is one KMS key per environment, used with `ED25519_SHA_512` and `MessageType RAW`. Each signature records its key ID and algorithm. Public keys are stored in Postgres by key ID, so verification never needs KMS. Rotation adds a new key, and old keys are disabled, never scheduled for deletion.
- **Claims.** This supports the AC 120-78B attributes. Never market it as "FAA-approved."

## Data ownership

- **Export is guaranteed.** A full CSV and PDF export of a pilot's logbook is available on every plan, including after a subscription lapses, and is offered before account deletion.
- **Account deletion** is available in every app, because the App Store requires in-app deletion. Google Play also requires a route from the web, so the site's account-deletion page links to the web app's. An idempotent, resumable job runs under the purge role, the only role that may delete from the append-only tables. It removes the pilot's rows, S3 objects, push tokens, WorkOS user and RevenueCat customer. It then tombstones the WorkOS user ID so a still-valid token can't recreate the account. Deleting the RevenueCat customer doesn't cancel a store subscription, so the app tells the pilot to cancel it in the store first. Point-in-time recovery and backup copies keep deleted data for at most 35 days.
- **Instructor records outlive student accounts.** Instructors must keep endorsement records for 3 years (14 CFR 61.189). Each signature an instructor gives is also stored as a self-contained instructor record: the signed payload and its signature. It stays verifiable after the student's rows are deleted, and a scheduled job purges it 3 years after the student's deletion.
- **Vendors get the minimum.** Under a zero-retention, no-training agreement, the vision LLM receives page images. Logfire and Sentry receive no request bodies or page content. Push payloads carry no personal data. WorkOS holds identity, RevenueCat purchase history and SES email addresses.

## Infrastructure (AWS)

| Concern | Choice |
| --- | --- |
| Containers | Three ECS Fargate services on ARM64 (Graviton), built from one image: the API, Taskiq workers, and the Taskiq scheduler. The scheduler runs `desiredCount 1` with deployment `minimumHealthyPercent 0` and `maximumPercent 100`, so two never run at once. A CloudWatch alarm fires when it has no running task, because the job sweeper depends on it. |
| Load balancer | Application Load Balancer at `api.tallyaloft.app`, with AWS WAF attached |
| DNS and certificates | Route 53 and ACM. The host map is in the [technical overview](../overview.md#domains). |
| Database | Amazon RDS for PostgreSQL, deletion protection on, 35-day point-in-time recovery, and AWS Backup copies to a separate account and region. Restores are rehearsed quarterly. |
| Cache and broker | Amazon ElastiCache for Valkey (cheaper than ElastiCache for Redis OSS) |
| Files and email | S3, SES, SNS and SQS in the same region |
| Signing keys | AWS KMS (Ed25519) |
| Secrets | AWS Secrets Manager, injected through ECS task `secrets` |
| Infrastructure as code | The `tally-aloft-infra` repository |
| Deploys | GitHub Actions with OIDC (no long-lived AWS keys): build and push the image to ECR, run the migration task, then update the three services. Development, staging and production each have their own settings, WorkOS environment and AWS resources. |

## Operations

- **Migrations** run as a one-off ECS task before each deploy. Old and new code run side by side during a rolling deploy, so schema changes are expand-then-contract: add, backfill, switch, then remove in a later release. Migrations set a `lock_timeout` and build indexes `CONCURRENTLY`.
- **Major Postgres upgrades** use RDS Blue/Green deployments to keep downtime short.

## Developer tooling

| Concern | Choice | Notes |
| --- | --- | --- |
| Packaging | uv | Lockfile committed; `uv sync --locked` in CI and images. Python 3.14 only (`requires-python = "==3.14.*"`). The build backend is `uv_build`, whose default src layout is `src/tally_aloft`. uv's own version is set once, in `[tool.uv] required-version`. Dependencies have lower bounds in `pyproject.toml`, and `uv.lock` holds the exact versions. |
| Local development | Docker Compose | Runs Postgres and Valkey. The API, worker and scheduler run with `uv run`. |
| Lint and format | Ruff | `select = ["ALL"]`, `line-length = 88`, preview off. Ruff's version is pinned in `uv.lock`, so an upgrade, and any rules it adds, is its own pull request.<br>• **Docstrings** are required except on public modules and packages (`D100` and `D104` ignored; a package's `__init__.py` is a module). A multi-line docstring's summary starts on the line after the opening quotes (`D213` over `D212`), with no blank line before a class docstring (`D211` over `D203`).<br>• **Formatter conflicts:** where a rule conflicts with the formatter, the formatter wins and the rule is ignored (`COM812`).<br>• **Other global ignores:** `CPY001`, because there is no copyright header policy.<br>• **Limits and reasons:** the global ignore list stays at 10 rules or fewer, and every global and per-file ignore has a reason comment. A tooling test checks both.<br>• **Per-file ignores:** `tests/` ignores all `D` rules, plus the rules tests need (`S101`, `PLR2004`, `S603`).<br>• **Runtime annotations:** pydantic models and settings are listed in `runtime-evaluated-base-classes`, so the `TC` rules never move an import that pydantic needs at runtime. |
| Type checking | Pyrefly | `preset = "strict"`, with `check-unannotated-defs = true`, over `src/` and `tests/`.<br>• **Error kinds:** every error kind that is off by default, or only a warning, is an error. One is off, with its reason on its line in `pyproject.toml`: `unused-call-result`, because discarding results such as `Path.write_text()`'s byte count is routine. Those settings are written out kind by kind, from Pyrefly's own list for the locked version, so a Pyrefly upgrade is the moment to check for new kinds.<br>• **Suppressions:** an inline suppression is `# pyrefly: ignore[<kind>]  # <reason>`, and `src/` holds 10 at most. A tooling test checks both. |
| Architecture rules | Import Linter | Contracts keep layers separated (for example, domain code never imports from API or infrastructure code). |
| Tests | pytest + pytest-asyncio | Async tests by default. `filterwarnings = ["error"]` and pytest's strict mode are on. `httpx` is the HTTP client for tests, both in process (`TestClient`, `ASGITransport`) and against running servers. |
| Test infrastructure | testcontainers | Real PostgreSQL (production's major version) and Valkey containers; no database mocks. |
| Test data | polyfactory | Typed factories built from the Pydantic models and domain dataclasses. |
| API compatibility | oasdiff | Breaking-change check on `openapi.json`, run through `oasdiff-action`. |
| CI | GitHub Actions | On every pull request: `ruff check`, `ruff format --check`, `pyrefly check`, `lint-imports`, `pytest`, the `openapi.json` freshness check and the oasdiff breaking-change check. |

## Deferred: offline-first sync

Version 1 requires a connection; every read and write goes through the API. If offline use is added later, the intended pattern is **PowerSync**:

- The PowerSync Service streams changes from Postgres through logical replication and keeps a local SQLite copy of each user's data on the device. Rules configured in the service decide which rows each user receives.
- Clients write to local SQLite. Queued writes upload to a FastAPI endpoint we own, which validates and applies them or rejects them. The server stays the single writer.
- Web and React Native SDKs are generally available. The service runs as PowerSync Cloud or self-hosted (Open Edition, Functional Source License).

To keep that option open, version 1 should:

- Use UUIDv7 primary keys that a client could generate, rather than database sequences.
- Keep write endpoints idempotent (already required; see the [client contract](../client-contract.md)). Offline writes can replay days later, so that path needs a longer key retention than 24 hours.
- Keep `updated_at` and soft-delete markers on synced tables.
- Treat signed entries as immutable; corrections create new versions.
- Plan for `rds.logical_replication = 1` on RDS, which needs a reboot to enable.
