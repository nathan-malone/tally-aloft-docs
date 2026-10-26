Backend · Structure · October 2026

# Tally Aloft — Backend Project Structure

The backend uses a **hexagonal (ports and adapters) architecture**. Business rules sit at the center and know nothing about FastAPI, Postgres or any vendor. Everything that touches the outside world is an adapter behind an interface (a "port"). Import Linter enforces the boundaries on every pull request. This document describes how the code is organized and where new things go. It defines the parts every version of the product needs (layers, wiring, authentication, jobs, the client contract) and describes, with examples, how product domains and features fit in as they're designed. The libraries named here are in [tech-stack.md](tech-stack.md).

## Why hexagonal

- **The rules are the product.** A logbook is a legal record, and its calculations are what pilots trust us with. Keeping rules pure makes them fast to test exhaustively and impossible to break with a database or framework change.
- **Vendors are replaceable.** Each external service sits behind one port, so swapping one means writing one adapter.
- **One set of use cases, many entry points.** HTTP requests, background jobs and webhooks call the same use cases.

The cost is extra code: database tables are mapped to domain objects, and each external service needs a port plus an adapter. We accept that for a legal record.

## Repository layout

```
tally-aloft-backend/
├── pyproject.toml            # dependencies, Ruff, Pyrefly, Import Linter config
├── uv.lock
├── alembic.ini
├── compose.yaml              # local Postgres and Valkey (added with their stories); profile `app` runs every process in containers
├── Dockerfile                # one image; API, worker and scheduler differ by command
├── .github/workflows/        # CI and deploy
├── openapi.json              # committed API schema; CI checks it's current
├── scripts/
│   └── export_openapi.py     # builds the app without connecting and writes openapi.json
├── docs/
├── migrations/               # Alembic env.py and versions/
├── src/tally_aloft/
│   ├── main.py               # API entry point
│   ├── worker.py             # Taskiq entry point: exposes broker and scheduler
│   ├── bootstrap/            # settings, app factory, wiring
│   ├── domain/
│   │   ├── shared/           # IDs, units, base errors
│   │   ├── accounts/         # the pilot
│   │   ├── logbook/          # flight entries
│   │   └── <domain>/         # one package per domain, added as the product grows
│   ├── application/
│   │   ├── ports/            # cross-cutting Protocols (Clock, JobQueue, Signer, …)
│   │   ├── registry.py       # UseCases: the only thing inbound adapters call
│   │   ├── shared/           # idempotency, jobs, pagination, non-domain errors
│   │   └── <domain>/         # that domain's use cases and ports, same names as domain/
│   └── adapters/
│       ├── inbound/
│       │   ├── http/         # routers, schemas, auth, client-version check, errors, webhooks
│       │   └── jobs/         # Taskiq task functions and schedules
│       └── outbound/
│           ├── postgres/     # tables, repositories, query services, unit of work
│           ├── redis/        # cache and rate limiting on Valkey, each behind a port
│           ├── taskiq/       # JobQueue
│           ├── aws/          # aiobotocore clients, one module per service (S3, SES, SQS, KMS)
│           ├── workos/       # TokenVerifier
│           └── <technology>/ # one package per vendor or technology
└── tests/
    ├── fakes/                # in-memory implementations of every port
    ├── contracts/            # one suite per stateful port, run against the fake and the real adapter
    ├── support/              # test-only apps, routes and helpers; never shipped
    ├── unit/domain/
    ├── unit/application/
    ├── unit/bootstrap/       # settings and wiring
    ├── integration/          # outbound adapters against real Postgres and Valkey
    ├── api/                  # HTTP requests through the full app
    ├── image/                # the built image and its commands, in Docker
    └── tooling/              # static checks of the repository's own configuration
```

The named packages are the ones the core needs. Everything shown as `<domain>` or `<technology>` is added when a feature needs it.

## The four layers

Dependencies point inward only: `main`/`worker` → `bootstrap` → `adapters` → `application` → `domain`.

### `domain/` — rules and records

Entities, value objects and pure functions, written as frozen dataclasses and plain Python. No I/O, no `async`, no third-party imports. A flight entry and its time columns, and the rules that check them, are typical domain code. Functions that depend on the date take `today` or `now` as an argument; they never read the clock.

### `application/` — use cases and ports

- **Use cases**, one module each, named by verb, such as `logbook/log_flight.py`. Each takes a command, a frozen dataclass that always carries `actor: PilotId`, checks that the actor may act on the data, and does its work inside one unit of work.
- **Scoped by construction.** Repository and query methods take the `PilotId` whose data they touch, so an unscoped read can't be written by accident. Access to another pilot's data, such as an instructor reviewing a student's entry, goes through a grant that the use case checks.
- **Unit of work.** Use cases receive a unit-of-work factory, never individual repositories: `async with self.uow() as uow:` exposes that domain's repositories (for example `uow.flights`, plus the shared `uow.idempotency` and `uow.jobs`), all on one database session and one transaction. Nothing inside a unit of work runs concurrently with `asyncio.gather`, because a session isn't safe to share between tasks.
- **Ports come in two kinds.**
  - **Domain ports** live with their domain, in `application/<domain>/ports.py`: that domain's repositories, read-side query services and unit-of-work Protocol.
  - **Cross-cutting ports** live in `application/ports/`, one module per capability, named by capability rather than technology. `Clock`, `JobQueue` and `TokenVerifier` exist from the start. Others, such as `ObjectStore`, `Signer`, `EmailSender` or `RateLimiter`, are added with the features that need them. These modules import only `domain.shared` and `application.shared`, and `ports/__init__.py` re-exports nothing.
  - Keeping domain ports out of `application/ports/` matters: Import Linter follows indirect imports, so a shared ports module that mentioned every domain would let any use case reach upward through it.
- **Reads.** List endpoints don't load aggregates. Query services, such as a `FlightQueries` for the logbook list, return read models one page at a time with keyset cursors (`Page[T]` in `application/shared`). The Postgres adapter implements them with SQLAlchemy Core.
  - Entries are append-only versions, so a head row per entry points at its current version, and lists read only current versions.
  - A list supports a fixed set of sort keys, each backed by an index, and filters as query parameters. The cursor encodes the sort value and the ID.
  - Totals are SQL aggregates over current versions, computed on request. Add caching only when measurements call for it. A career logbook holds up to about 20,000 entries.
- **`registry.py`** defines `UseCases`, a typed collection of every use case. Bootstrap builds it; inbound adapters read it. Use cases never import it.

### `adapters/` — the outside world

- **Inbound** adapters translate an outside event into a use-case call.
  - `http/` holds FastAPI routers grouped by domain, Pydantic request and response schemas, the authentication dependency, the client-version middleware, error mapping and webhook receivers. Routers get `UseCases` from `request.app.state`.
  - `jobs/` holds Taskiq tasks as plain async functions plus the schedule list. Tasks read `UseCases` from Taskiq's worker state through `TaskiqDepends`.
- **Outbound** adapters implement ports with a specific technology and are named after it: `PostgresFlightRepository`, `S3ObjectStore`, `KmsSigner`. SQLAlchemy table models live only in `outbound/postgres/tables/`, with mappers to and from domain objects. One `PostgresUnitOfWork` implements every domain's unit-of-work Protocol. Vendor-specific schemas, such as a vendor's webhook payload or response models, stay inside that vendor's adapter. Adapters that do CPU-bound work run it in a thread or process pool, never on the event loop.
- Adapters never import each other. Shared resources, such as the aiohttp `ClientSession` and the Taskiq broker, are created by bootstrap and passed into adapter constructors.

### `bootstrap/` — wiring

- `settings.py` is the only module that reads configuration.
  - **Source.** Settings come from environment variables only, all prefixed `TALLY_`. Unknown `TALLY_` variables are ignored.
  - **Environments.** `TALLY_ENVIRONMENT` is one of `development`, `test`, `staging` or `production`.
  - **Formats.** Durations are integer seconds, lists are comma-separated, and maps are JSON.
  - **`.env` files.** For local development, a git-ignored `.env` file, copied from the committed `.env.example`, is loaded by the entry command (`uv run --env-file .env …`), never by `Settings`, and the image contains none.
  - **Startup errors.** Entry points call `load_settings()`. On a missing or invalid variable, it exits with one line that names each failing variable and its error type. The line is built from each error's location and type, never from `str(ValidationError)`, so no value appears in it. Secret settings use `SecretStr`.
  - **Test and export profiles.** `Settings.for_tests(**values)` is the test suite's profile, and `Settings.for_schema_export()` needs no variables or secrets. Neither reads the process environment, so a developer's shell can't change them.
- `app.py` defines `create_app(settings, overrides=None)`. It registers routers and middleware and nothing else; it opens no connection and builds no adapter, because an aiohttp `ClientSession` can only be created while the event loop is running. The FastAPI lifespan then creates the engine, the HTTP session and the broker connection (`broker.startup()`), calls `container.build_use_cases(...)`, and sets `app.state.use_cases`. API tests pass fakes through `overrides`.
- `container.py` builds every adapter and the `UseCases` registry from those resources. It also defines `register_tasks(broker)`, which registers the task functions and schedules from `adapters/inbound/jobs` (`broker.register_task`).
  - `worker.py` calls `register_tasks` when it's imported, not at startup. The scheduler process never runs worker startup handlers, so registering at startup would leave it with no schedules.
  - Only `UseCases` is set at startup, in a `WORKER_STARTUP` handler that puts it into Taskiq's state.
- `main.py` and `worker.py` call bootstrap and nothing else. `worker.py` exposes both `broker` (for worker processes) and `scheduler` (for the single scheduler service).

## Domains

A domain is a business area with its own rules and data. Each domain has a package in `domain/` and one with the same name in `application/`, and a router in `adapters/inbound/http/` if clients call it.

Two domains exist in every version of the product:

- **`accounts`**: the pilot. Identity lives in WorkOS. `accounts` holds what we know about the pilot under an internal `PilotId` (a UUIDv7), with the WorkOS user ID as a unique attribute, so replacing the identity provider doesn't touch other domains.
- **`logbook`**: flight entries and their edit history.

Other domains are added as the product is designed. When one is proposed:

- **Give it a level.** A domain may use domains on lower levels. A domain's level is one above the highest domain it depends on, so a domain that depends only on `shared` sits on level 1, and a domain that reads flight entries sits above `logbook`. A domain that two others both need sits below both of them.
- **Keep same-level domains apart.** Same-level domains never import each other's rules or use cases. A use case may read a same-level domain's types, but if it needs that domain's behavior, the two aren't on the same level: move one up.
- **Reference lower domains, never higher ones.** When a higher domain needs to point at lower-level data, it stores the lower record's ID. The lower domain never learns about the higher one. For example, anything that attests to a logbook entry references a specific entry version, so `logbook` never needs to know what refers to it.
- **Place cross-domain operations and read models at the top.** An operation or read model that spans several domains lives in the highest domain it touches. A logbook list that shows signature status, for example, belongs to the domain that owns signatures. If it fits none of them, it gets its own domain above them all. Exporting or deleting a pilot's data, which touches every domain holding pilot data, is an example.
- **Split by rules, not screens.** One screen can draw on several domains, and one domain can serve many screens.
- **Update the contracts.** Add the domain to both level contracts in `pyproject.toml` (see Enforced boundaries) in the same pull request that creates its packages.

The levels as they stand today:

| Level | Domains |
| --- | --- |
| 1 | `accounts`, `logbook` (each depends only on `shared`; an entry refers to its pilot by `PilotId`) |
| 0 | `shared` (IDs, units, base errors) |

## Authentication

The HTTP authentication dependency calls `UseCases.authenticate`. It works in three steps:

1. **Verify the token.** It checks the access token through the `TokenVerifier` port. With the key set cached, this is CPU work only.
2. **Find the pilot.** It maps the WorkOS user ID to the pilot's `PilotId`. The mapping is cached per process for 5 minutes. On a cache miss, it reads the mapping, or creates it with an upsert the first time a user appears. A deleted account's WorkOS user ID is tombstoned, so it can't be recreated.
3. **Pass the actor on.** Routers put the `PilotId` into every command as `actor`. Use cases, not routers, decide whether that actor may read or change the data.

## Background jobs

- **The job row is the outbox.** Work that can't run inside the database transaction (generating a file, enqueuing a task, calling a slow vendor) is recorded in it. The use case inserts a `jobs` row with status `queued` in its own transaction. After commit, it dispatches the task through the `JobQueue` port.
- **Tasks claim before they work.** Delivery is at least once, so a task first claims its job:

  ```sql
  UPDATE jobs
  SET status = 'running', attempts = attempts + 1, lease_until = now() + :lease
  WHERE id = :id
    AND (status = 'queued' OR (status = 'running' AND lease_until < now()))
  RETURNING …
  ```

  If no row comes back, another worker holds the job or it's finished, and the task stops. Long tasks renew their lease. A task that calls a vendor records the call on the job row right after it, so a re-run skips it.
- **The sweeper recovers stalls.** A scheduled job re-dispatches `queued` jobs whose `last_dispatched_at` is over 5 minutes old and updates that column. It also re-dispatches `running` jobs whose lease has expired. A job that reaches `max_attempts` is marked `failed`, which also catches workers killed without raising (out of memory, for example). Clients see `failed` through `GET /jobs/{id}`.
- **Dispatch by name.** `TaskiqJobQueue` kicks tasks by name (`AsyncKicker(task_name, broker, labels)`), so it never imports `adapters/inbound/jobs`.
- **Inbound events from AWS.** Scheduled jobs that read queued events, such as SES bounce and complaint notices from SQS, call a use case that reads them through a port.

## Client contract in the code

The [client contract](../client-contract.md) sets the rules for the web and mobile clients. This is where each rule lives.

| Rule | Where | How |
| --- | --- | --- |
| Operation IDs | `adapters/inbound/http/operation_ids.py` | Each router is created with one tag: its domain (`tags=["logbook"]`), or for shared routes its capability (`tags=["jobs"]`). Endpoint functions are named by verb (`list_flights`). A `generate_unique_id_function` builds `logbook_list_flights`; `create_app` passes it to `FastAPI(...)`. Webhook and health routes use `include_in_schema=False`. |
| Schema export | `scripts/export_openapi.py` | Builds the app with `Settings.for_schema_export()`, writes `openapi.json`, and fails on a route without a tag or a duplicate operation ID. CI re-runs it and fails on a diff, then runs oasdiff against every supported client release's schema. |
| Client versions | `adapters/inbound/http/client_version.py` | Middleware reads `X-Client-Platform` and `X-Client-Version` and compares them with a `ClientVersionPolicy` (minimum version per platform) that bootstrap builds from settings. It returns the `400` `client_version_unsupported` response itself, because exceptions raised in middleware don't reach FastAPI's exception handlers. Webhook routes are exempt. |
| Idempotency | `application/shared/idempotency.py`; `uow.idempotency`; table `idempotency_keys` | See [Idempotency](#idempotency) below. |
| Concurrent edits | The use case; the entry's head row | The use case locks the head row (`SELECT … FOR UPDATE`), compares `base_version` with the current version, and raises `version_conflict` if they differ. |
| Job status | `application/shared/get_job_status.py`, route `GET /jobs/{id}` (tag `jobs`) | Returns status and, when done, a presigned download URL for any file the job produced. |
| Error shape | `domain/shared/errors.py`, `application/shared/errors.py`, `adapters/inbound/http/errors.py` | Every domain error declares a stable `code`. `DomainError` and `ApplicationError` both derive from `CodedError` in `domain/shared/errors.py`, which checks at class definition that each concrete error sets its own string `code` (a `TypeError` otherwise), so no two errors share one by inheritance. A base that only groups other errors is declared with `abstract=True` and needs none. Errors that aren't about the domain (authentication, entitlement, idempotency reuse, rate limits, unsupported client) live in `application/shared/errors.py`, also with codes. The HTTP adapter maps each to a status and the shared body `{code, message, details}`. It overrides FastAPI's `RequestValidationError` handler and 422 schema so validation errors use the same body, and routers declare their error responses so the schema documents them. |
| CORS | `bootstrap/app.py` | Starlette's `CORSMiddleware`, the outermost user middleware that can produce a response, so it also covers rejected requests. Only Uvicorn's proxy-headers wrapper, which never responds, sits outside it among the user middleware. `create_app` passes its user middleware to `FastAPI(middleware=[...])` as one list, outermost first. Starlette always wraps the whole list in its own `ServerErrorMiddleware`, whose 500 responses carry no CORS headers, so unhandled errors are caught by a middleware inside CORS. Origins come from settings, and the headers are those listed in the client contract. |

### Idempotency

1. **The router passes the key in.** Write routers take the key as a required `Header(alias="Idempotency-Key")` parameter, so it appears in `openapi.json`. They put it in the command, along with a hash of the request body.
2. **The use case wraps its change.** Inside its unit of work, it calls `run_once(uow, operation, key, request_hash, action)`.
3. **`run_once` inserts the key row first.** It runs `INSERT … ON CONFLICT DO NOTHING RETURNING` on (pilot, operation, key).
   - **A row comes back.** This is the first request. The action runs, and its result (status and body, or a reference for large results) is written to the key row before commit.
   - **No row comes back.** A duplicate exists. Under READ COMMITTED, the insert first waits for a concurrent first request to finish. `run_once` then reads the stored row and returns its result. If the hash differs, it raises `idempotency_key_reused` instead.
4. **A failed action stores nothing.** If the action raises, the transaction rolls back and takes the key row with it, so a retry runs fresh.
5. **Old keys are cleaned up.** A scheduled job deletes keys older than 24 hours.

## Migrations and reference data

- Alembic's `env.py` targets the table models' `MetaData`, with a constraint `naming_convention` so constraint names are stable. How migrations are deployed is in [tech-stack.md](tech-stack.md#operations).
- Bulk external datasets, such as airport or aircraft registries, load through a bulk-writer port, not row by row through repositories.

## Where things go

| Adding… | Goes in |
| --- | --- |
| A business rule or calculation | `domain/<domain>/` |
| A new domain | Packages in `domain/` and `application/`, a level in both domain contracts (see Domains) |
| A new operation a client can perform | `application/<domain>/<verb>.py`, then add it to `UseCases` |
| A repository or query service | Its Protocol in `application/<domain>/ports.py`; the Postgres implementation in `adapters/outbound/postgres/`; a fake in `tests/fakes/` |
| A list endpoint | A query service returning `Page[T]`, not a repository |
| An endpoint | `adapters/inbound/http/<domain>.py`, calling a use case |
| A write endpoint | Also takes the `Idempotency-Key` header; the use case wraps its change in `run_once`. An update also takes `base_version`. |
| A background or scheduled job | A job row from the use case; the task in `adapters/inbound/jobs/`, registered in bootstrap |
| A new external service | A port in `application/ports/`, an adapter in `adapters/outbound/<technology>/`, wiring in `bootstrap/container.py`, and a fake in `tests/fakes/`. Add a suite in `tests/contracts/` when the fake holds state. |
| A webhook from a vendor | A receiver in `adapters/inbound/http/webhooks/` that verifies the signature, deduplicates on the event ID and calls a use case |
| A database table or column | `adapters/outbound/postgres/tables/`, then an expand-then-contract Alembic migration |
| A setting | `bootstrap/settings.py`, passed to the adapter that needs it |
| A new error a client can act on | A domain or application error with a new `code`, mapped in `adapters/inbound/http/errors.py` |
| A change to an existing endpoint | Add, don't change (see Breaking changes in the [client contract](../client-contract.md)) |

## Testing

| Suite | Tests | Uses |
| --- | --- | --- |
| `unit/domain` | Rules and calculations | Plain pytest; no fixtures needed |
| `unit/application` | Use cases, including authorization, with a denied case for each use case | Fakes from `tests/fakes/` |
| `contracts` | That each stateful fake (repositories, object store, job queue) behaves like its real adapter | One suite per port, parametrized over the fake and the real adapter |
| `integration` | Postgres and Valkey adapters, migrations, and the concurrency guarantees: <br>• two concurrent requests with one key produce one write <br>• a duplicate or late delivery doesn't run a job twice <br>• expired leases are recovered <br>• the sweeper re-dispatches a job that was never dispatched <br>• the application role can't update or delete audit rows | testcontainers. Migrations run as the owner role; tests connect as the application role. |
| `api` | Routing, auth, error mapping, schemas, client-version rejection, idempotent replay, version conflicts, and the API process started on the host | The full app, with outbound adapters replaced by fakes through `overrides` |
| `image` | The release image: its user, size and contents, proxy trust, and shutdown on `SIGTERM` | Docker, marked `container`. Tests run the image's own default command and swap only the app path for `tests.support.probe_app:app`, with `tests/` mounted read-only, so test-only routes reach the real server flags while the image ships no tests. CI passes its built image in `BACKEND_TEST_IMAGE`. |
| `tooling` | The repository's configuration: the Ruff and Pyrefly policies, the Import Linter contracts, the lockfile, the `Dockerfile`, `compose.yaml`, the CI workflow and the settings list | Plain pytest over the files, plus the tools themselves on probe inputs |

`tests/fakes/__init__.py` assigns each fake to a variable typed as its Protocol, so Pyrefly fails when a fake drifts from its port. Vendor adapters get opt-in tests against sandbox accounts, outside the default pull-request run.

## Enforced boundaries

These Import Linter contracts in `pyproject.toml` run in CI. A pull request that breaks one fails. The two domain-level contracts list the domains that exist today and grow as domains are added.

```toml
[tool.importlinter]
root_package = "tally_aloft"
include_external_packages = true

[[tool.importlinter.contracts]]
name = "Dependencies point inward"
type = "layers"
layers = [
    "tally_aloft.main | tally_aloft.worker",
    "tally_aloft.bootstrap",
    "tally_aloft.adapters",
    "tally_aloft.application",
    "tally_aloft.domain",
]

# Levels inside each package. Every domain has a package in both.
# Same-level domains go on one line, separated by " | ".
[[tool.importlinter.contracts]]
name = "Domain levels within domain/ and application/"
type = "layers"
containers = ["tally_aloft.domain", "tally_aloft.application"]
layers = [
    "logbook | accounts",
    "shared",
]
exhaustive = true
exhaustive_ignores = ["ports", "registry"]

# Levels across the two packages, so a use case can't reach up into a
# higher domain's rules. Each line pairs a domain with its use cases;
# same-level domains join the same line with " : ".
[[tool.importlinter.contracts]]
name = "Domain levels across domain/ and application/"
type = "layers"
layers = [
    "tally_aloft.application.logbook : tally_aloft.domain.logbook : tally_aloft.application.accounts : tally_aloft.domain.accounts",
    "tally_aloft.application.shared : tally_aloft.application.ports : tally_aloft.domain.shared",
]

[[tool.importlinter.contracts]]
name = "Adapters are independent"
type = "independence"
modules = [
    "tally_aloft.adapters.inbound.*",
    "tally_aloft.adapters.outbound.*",
]

[[tool.importlinter.contracts]]
name = "Domain and application have no framework or vendor code"
type = "forbidden"
source_modules = ["tally_aloft.domain", "tally_aloft.application"]
forbidden_modules = [
    "fastapi", "starlette", "uvicorn", "pydantic", "pydantic_settings",
    "sqlalchemy", "asyncpg", "alembic", "redis", "taskiq", "taskiq_redis",
    "aiohttp", "aiobotocore", "pydantic_ai", "typst", "cryptography",
    "rfc8785", "jwt", "logfire", "opentelemetry",
]
```

How the two level contracts divide the work:

- **Within each package**, same-level domains are fully independent. The contract is exhaustive, so a domain package missing from it fails CI.
- **Across packages**, a use case can't reach up into a higher domain's rules. Import Linter can't mix independent (`|`) and connected (`:`) modules on one line, so this contract can't stop a use case from reading a same-level domain's types; review covers that rule. The contract also has no containers, so it can't be exhaustive: check that each new domain was added to it.
- **Ports sit on the bottom line** of the cross-package contract, so a port module that imports a domain's rules fails CI. `registry.py` isn't listed, and nothing in a level imports it.

When a new vendor library is adopted, add it to the forbidden list. Import Linter fails on a forbidden package that isn't installed, so the list in `pyproject.toml` holds only installed packages, including transitive ones such as `starlette`, `aiohttp` and `opentelemetry`. The block above is the target list.

## Conventions

- Package root is `src/tally_aloft`; tests import it as an installed package.
- Use cases are `async`; domain code is synchronous.
- Commands and results are frozen dataclasses; Pydantic stays in adapters and settings.
- Use cases raise domain or application errors. Only `adapters/inbound/http` turns them into HTTP responses.
- Logging inside `domain` and `application` uses the standard `logging` module; Logfire instrumentation is configured in bootstrap and on adapters.
