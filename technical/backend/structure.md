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
├── pyproject.toml            # project metadata, dependencies and uv settings
├── uv.lock
├── ruff.toml                 # Ruff lint and format settings
├── pyrefly.toml              # Pyrefly type-check settings
├── pytest.toml               # pytest settings
├── .importlinter             # Import Linter contracts (INI)
├── package.json              # npm developer tools (aislop); pnpm-lock.yaml locks them
├── pnpm-workspace.yaml       # pnpm settings: the 7-day minimum release age
├── .aislop/config.yml        # the AI-slop check
├── Makefile                  # every developer and CI command
├── alembic.ini               # Alembic settings and the Ruff hooks for new revisions
├── compose.yaml              # local Postgres and Valkey; profile `app` runs the migrations and every process in containers
├── compose.e2e.yaml          # swaps the worker for the test worker, built from the `dev` stage
├── compose/
│   └── postgres-init.sql     # creates the two login roles in local and test Postgres
├── Dockerfile                # one image; API, worker and scheduler differ by command; a `dev` stage adds tests/
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
- **Unit of work.** Use cases receive a unit-of-work factory, never individual repositories: `async with self.uow() as uow:` exposes that domain's repositories (for example `uow.flights`, plus the shared `uow.idempotency` and `uow.jobs`), all on one database session and one transaction. It commits when the block exits cleanly and rolls back when it raises, cancellation included. Every domain's unit-of-work Protocol extends `SharedUnitOfWork` in `application/shared/unit_of_work.py`, whose `__aenter__` returns `Self`, so `uow` keeps the domain's type; the shared, pilot-scoped repositories join it as they are built. Unscoped maintenance repositories, such as the jobs' `job_runner`, sit on their own Protocol extending it (`JobRunnerUnitOfWork` in `application/shared/jobs/ports.py`), which only the use cases that need them take, so a feature use case can't reach another pilot's data through its unit of work. Nothing inside a unit of work runs concurrently with `asyncio.gather`, because a session isn't safe to share between tasks.
- **Ports come in two kinds.**
  - **Domain ports** live with their domain, in `application/<domain>/ports.py`: that domain's repositories, read-side query services and unit-of-work Protocol.
  - **Cross-cutting ports** live in `application/ports/`, one module per capability, named by capability rather than technology. `Clock`, `JobQueue`, `TokenVerifier`, `Cache` and `DependencyCheck` exist from the start. `Clock` (`ports/clock.py`) has one method, `now()`, which returns an aware UTC `datetime`; `SystemClock` (`outbound/system/`) is the one place production code reads the system clock, and Ruff's banned-API rule fails any other read of `datetime.now`, `datetime.utcnow`, `date.today` or `time.time` in `src/`. Job leases are the one exception: they compare with the database's `now()` in SQL. `JobQueue` (`ports/job_queue.py`) has `dispatch(job_id, kind_name)`, which sends a recorded job's ID to the task registered under its kind's name; `TaskiqJobQueue` (`outbound/taskiq/`) implements it. `Cache` (`ports/cache.py`) stores `bytes` under string keys with `get`, `set` and `delete`. `set` takes a required time to live of at least 1 millisecond, and a shorter one raises `ValueError`, a programming error that isn't mapped to HTTP. A store that fails or stalls raises the port's own `CacheUnavailableError`, so use cases never see the client's errors. Cached personal data lasts until its TTL runs out or its key is deleted, so keep such TTLs short, or delete the key with the data. `ValkeyCache` (`outbound/redis/`) prefixes every key with `cache:`, keeping it apart from the broker's keys, and sets the TTL in milliseconds. `DependencyCheck` (`ports/health.py`) has a `name` and an async `check()` that raises when its dependency doesn't work; each data store's outbound adapter supplies one, and `UseCases.check_readiness` runs them for `GET /readyz`. Others, such as `ObjectStore`, `Signer`, `EmailSender` or `RateLimiter`, are added with the features that need them. These modules import only `domain.shared` and `application.shared`, and `ports/__init__.py` re-exports nothing.
  - Keeping domain ports out of `application/ports/` matters: Import Linter follows indirect imports, so a shared ports module that mentioned every domain would let any use case reach upward through it.
- **Reads.** List endpoints don't load aggregates. Query services, such as a `FlightQueries` for the logbook list, return read models one page at a time with keyset cursors (`Page[T]` in `application/shared`). The Postgres adapter implements them with SQLAlchemy Core.
  - Entries are append-only versions, so a head row per entry points at its current version, and lists read only current versions.
  - A list supports a fixed set of sort keys, each backed by an index, and filters as query parameters. The cursor encodes the sort value and the ID.
  - Totals are SQL aggregates over current versions, computed on request. Add caching only when measurements call for it. A career logbook holds up to about 20,000 entries.
- **`registry.py`** defines `UseCases`, a typed collection of every use case. Bootstrap builds it; inbound adapters read it. Use cases never import it.

### `adapters/` — the outside world

- **Inbound** adapters translate an outside event into a use-case call.
  - `http/` holds FastAPI routers grouped by domain, Pydantic request and response schemas, the authentication dependency, the client-version middleware, error mapping and webhook receivers. Routers get `UseCases` from `request.app.state` through one typed dependency, the `UseCasesDep` parameter annotation in `http/dependencies.py`.
  - `jobs/` holds Taskiq tasks as plain async functions plus the schedule list. Tasks read `UseCases` from Taskiq's worker state through `TaskiqDepends`.
- **Outbound** adapters implement ports with a specific technology and are named after it: `PostgresFlightRepository`, `S3ObjectStore`, `KmsSigner`. SQLAlchemy table models live only in `outbound/postgres/tables/`, with mappers to and from domain objects. One `PostgresUnitOfWork` implements every domain's unit-of-work Protocol. Each instance serves one `async with`, entering SQLAlchemy's `async_sessionmaker.begin()`, and builds its repositories on its `session` when entered. `unit_of_work_factory(engine)` returns the factory, making a new one on each call, that bootstrap passes to each use case that takes a unit of work; the container builds it once, on the application's engine. Vendor-specific schemas, such as a vendor's webhook payload or response models, stay inside that vendor's adapter. Adapters that do CPU-bound work run it in a thread or process pool, never on the event loop.
- Adapters never import each other. Shared resources, such as the aiohttp `ClientSession` and the Taskiq broker, are created by bootstrap and passed into adapter constructors.

### `bootstrap/` — wiring

- `settings.py` is the only module that reads configuration.
  - **Source.** Settings come from environment variables only, all prefixed `TALLY_`. Unknown `TALLY_` variables are ignored.
  - **Environments.** `TALLY_ENVIRONMENT` is one of `development`, `test`, `staging` or `production`.
  - **Formats.** Durations are integer seconds, lists are comma-separated, and maps are JSON.
  - **`.env` files.** For local development, a git-ignored `.env` file, copied from the committed `.env.example`, is loaded by the entry command (`uv run --env-file .env …`), never by `Settings`, and the image contains none.
  - **Startup errors.** Entry points call `load_settings()`. On a missing or invalid variable, it exits with one line that names each failing variable and its error type. The line is built from each error's location and type, never from `str(ValidationError)`, so no value appears in it. Secret settings use `SecretStr`.
  - **Connection URLs.** `connection_urls.py` holds the checks `settings.py` runs on the data stores' URLs, `check_database_url` and `check_valkey_url`, each told whether the process is deployed. Each parses with its client's own parser, so settings accept exactly what the client will, and reads no configuration itself.
  - **Migration settings.** `MigrationSettings`, read by `load_migration_settings()`, is the migration task's own model: the owner role's database URL and the lock timeout, in `TALLY_MIGRATION_*` variables, and `TALLY_ENVIRONMENT`, which decides whether the URL must verify the server, as for the services. Only Alembic's `env.py` reads it, and `Settings` has no field for any `TALLY_MIGRATION_*` variable, so the services never hold the owner's credentials.
  - **Test and export profiles.** `Settings.for_tests(**values)` and `MigrationSettings.for_tests(**values)` are the test suite's profiles, and `Settings.for_schema_export()` needs no variables or secrets; the required database and Valkey URLs are filled with placeholders that refuse connections. None of them reads the process environment, so a developer's shell can't change them.
- `broker.py` defines `build_broker(settings)`, the one factory for the Taskiq `RedisStreamBroker`, which connects only when first used.
  - **Names.** The stream is `broker:tasks` and the consumer group `broker:workers`, apart from the cache's `cache:` keys; taskiq-redis names its reclaim lock `autoclaim:broker:workers:broker:tasks` itself.
  - **The group.** Only workers create it, as their broker starts, and it starts at the stream's first message (`0`), not at `$`, so messages the API added before any worker started are still delivered. The same rule means a group created over a stream that already holds messages delivers every one of them again, so never rename or recreate the group over a live stream without trimming or deleting the stream first: job tasks shrug the replay off through the claim, but scheduled tasks would run once per old message.
  - **Limits.** The stream is trimmed approximately to `TALLY_BROKER_STREAM_MAXLEN`. `idle_timeout` comes from `TALLY_BROKER_IDLE_TIMEOUT`, converted to the milliseconds taskiq-redis counts in. A worker reads one message at a time (`xread_count=1`) and reclaims one idle message after each new one (`unacknowledged_batch_size=1`), so it holds only what it can start. Its reads block for 2 seconds, below the connection's 5-second socket timeout.
  - **Connections.** The pool takes `ConnectionOptions` from `outbound/redis/client.py`, the same timeouts and retry as the cache's client.
- `resources.py` defines `open_resources(settings, broker=None)`, the one place that opens the shared resources: the database engine, the cache's Valkey client and the broker now, and the HTTP session as it is adopted. It is an async context manager built on an `AsyncExitStack`, so the resources close in reverse order on exit, even when the body raises. It yields a frozen `Resources`, which the API's lifespan and the worker's startup both pass to `build_use_cases`, so the two processes open the same things the same way. The API passes no broker, so one is built and shut down here, and never connected at startup: the API kicks with `XADD`, which needs no consumer group, so it starts while Valkey is down. A worker passes its own broker, which Taskiq starts and shuts down.
- `app.py` defines `create_app(settings, overrides=None)`. It registers routers and middleware and nothing else; it opens no connection and builds no adapter, because an aiohttp `ClientSession` can only be created while the event loop is running. The FastAPI lifespan then enters `open_resources(settings)`, calls `container.build_use_cases(settings, resources, overrides)`, and sets `app.state.use_cases`. API tests pass fakes through `overrides`.
- `container.py` builds every adapter and the `UseCases` registry from those resources. It also holds `TASKS`, the production task list, and defines `register_tasks(broker, tasks)`, which registers each `TaskDefinition` (`adapters/inbound/jobs/tasks.py`: a name, a function, a timeout, and the job kind it runs, if any) under its name with its timeout as Taskiq's `timeout` label. Before registering anything it fails when two tasks share a name, which Taskiq would silently resolve to the last, and when `TALLY_BROKER_IDLE_TIMEOUT` isn't greater than the longest task timeout, naming both values: another worker could otherwise reclaim the message of a task that is still running.
- `worker.py` (in `bootstrap/`) defines `create_worker(settings, tasks)`, which builds the broker, adds the labels middleware (see [Background jobs](#background-jobs)), wires the worker's lifecycle and calls `register_tasks`.
  - The top-level `worker.py` calls it when it's imported, not at startup, with `TASKS`. The scheduler process never runs worker startup handlers, so registering at startup would leave it with no schedules. The test worker, `tests/support/worker.py`, calls it with the test tasks.
  - Only `UseCases` is set at startup, in a `WORKER_STARTUP` handler that enters `open_resources(settings, broker)` and puts `UseCases` into Taskiq's state. A `WORKER_SHUTDOWN` handler closes the resources. Tasks read `UseCases` through `UseCasesDep` in `adapters/inbound/jobs/dependencies.py`. Taskiq reads task annotations at run time, so task modules import their annotation types outside `TYPE_CHECKING`.
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
- **Update the contracts.** Add the domain to both level contracts in `.importlinter` (see Enforced boundaries) in the same pull request that creates its packages.

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

- **The job row is the outbox.** Work that can't run inside the database transaction (generating a file, enqueuing a task, calling a slow vendor) is recorded in it. The use case inserts a `jobs` row with status `queued` in its own transaction, through `uow.jobs.record(actor, kind, payload)`. After its `async with` block has committed, it calls `dispatch_safely(queue, jobs)` (`application/shared/jobs/dispatch.py`), which dispatches each job through the `JobQueue` port, one at a time, each under a 5-second limit. A dispatch that fails or times out is logged with the job's ID, kind and the error's type, never raised: the job is already `queued`, and the sweeper re-dispatches it, through the same function. The message carries only the job's ID; the row holds the payload.
- **Statuses.** A job is `queued`, `running`, `succeeded` or `failed`.
- **Job kinds.** Each kind of job is a `JobKind` constant in the application layer (`application/shared/jobs/kinds.py`): a stable `name`, a payload codec and a result codec, `max_attempts` (default 5), a `lease` (default 5 minutes) and a `time_limit` (default 10 minutes). Its Taskiq task is registered under that exact name, so moving a module can't break stored jobs.
- **Tasks claim before they work.** Delivery is at least once, so a task first claims its job:

  ```sql
  UPDATE jobs
  SET status = 'running', attempts = attempts + 1, lease_until = now() + :lease
  WHERE id = :id
    AND kind = :kind
    AND attempts < max_attempts
    AND (status = 'queued' OR (status = 'running' AND lease_until < now()))
  RETURNING …
  ```

  If no row comes back, another worker holds the job, it's finished or exhausted, or it belongs to another kind, and the task stops. The kind condition stops a message carrying another kind's job ID from running it with the wrong codecs; that job stays `queued` for the sweeper. A task that calls a vendor records the call on the job row right after it, so a re-run skips it.
- **Fencing.** Every later write to the row is fenced on the `attempts` value the claim set: renewing the lease, completing, failing the attempt, and any progress a body records update the row only `WHERE status = 'running' AND attempts = :claimed_attempts`. When nothing is updated, another claim has taken the job over, and the task writes nothing more. Completing sets `succeeded`, stores the result and clears any earlier attempt's `error_code`, so `error_code` is set only on a job whose last attempt failed. Failing an attempt records an `error_code` and sets `failed` once `attempts` has reached `max_attempts`, `queued` otherwise; the retry waits for the sweeper.
- **Lifecycle use cases.** Claim, renew lease, complete and fail attempt are use cases on `UseCases` (`application/shared/jobs/`), each in its own unit of work, because the task wrapper is an inbound adapter and can't touch the database. They work through `uow.job_runner`, the unscoped repository, which only they and maintenance use cases reach: it is on `JobRunnerUnitOfWork`, not on `SharedUnitOfWork` (see "application/"). `uow.jobs` is pilot-scoped and on every unit of work.
- **Time.** Leases compare with the database's `now()` in SQL, the one exception to reading time through `Clock`, so every worker uses one clock.
- **The task wrapper.** `job_task(kind, body)` (`adapters/inbound/jobs/tasks.py`) builds a job kind's task, which calls `run_job` (`run_job.py`) with the job's ID. The wrapper only orchestrates the lifecycle use cases:
  1. It claims the job. If the claim returns nothing, it stops without running the body.
  2. It runs the body under `asyncio.timeout(kind.time_limit)`, after decoding the payload with the kind's codec, and encodes the result. Decoding happens after the claim, so a payload that won't decode fails the attempt like any body error.
  3. Meanwhile, a separate asyncio task renews the lease every half lease, each renewal in its own unit of work, never the body's session. It keeps a local deadline: the start of the claim or of the last successful renewal, plus nine tenths of the lease. A renewal that raises is retried after a tenth of the lease. When a renewal finds the job taken over, or the deadline passes without one, the wrapper cancels the body, logs `lease_lost` and writes nothing, so a body never outlives the lease the database holds.
  4. Otherwise it completes the job with the result, or fails the attempt with the body's error `code` for a domain or application error, `timeout` when the time limit expired, or `unhandled_error` for anything else, a `TimeoutError` the body raised itself included. A completion or failure that matches no row logs `lease_lost`.

  The wrapper enforces the time limit around the body only. Taskiq's `timeout`, the time limit plus 30 seconds, applies to the whole task and only stops a wrapper that hangs. The margin covers the claim and the outcome's write, each of which may wait for a pooled connection and then run up to the statement timeout, so it holds only while the pool has room for every running task's renewal and write (see the worker command's **Concurrency** in [tech-stack.md](tech-stack.md#operations)). A wrapper that is cancelled, by that backstop or otherwise, cancels the body and the renewal before it waits for either, because Taskiq cancels through anyio, which cancels again on every loop turn; it writes nothing, and the row stays `running` for the sweeper. If writing the outcome raises, as when the database is down, the error propagates: Taskiq logs it and acknowledges the message, and the row stays `running` until its lease expires. At worker shutdown Taskiq doesn't cancel running tasks: one still running when the process exits is abandoned, its message unacknowledged.
- **Job bodies.** A body is the main extension point for every job: an async use case on `UseCases` that receives a `ClaimedJob(id, pilot_id, kind, attempts, payload)`, with `kind` the kind's name and `payload` decoded, and returns the result, which the kind's result codec encodes. Each `JobKind` carries its payload and result codecs, so bodies stay typed under strict Pyrefly. A body that writes progress to its own job row fences the write on the claimed `attempts`. A body that holds a unit of work while it runs holds a pooled connection beside its renewal's, so a long one should commit in short transactions. `job_task`'s `body` argument picks the use case from `UseCases`.
- **The sweeper recovers stalls.** A scheduled job re-dispatches `queued` jobs whose `last_dispatched_at` is over 5 minutes old and updates that column. It also re-dispatches `running` jobs whose lease has expired. A job that reaches `max_attempts` is marked `failed`, which also catches workers killed without raising (out of memory, for example). Clients see `failed` through `GET /jobs/{id}`.
- **Dispatch by name.** `TaskiqJobQueue` kicks tasks by name (`AsyncKicker(task_name, broker, labels)`), so it never imports `adapters/inbound/jobs`. Taskiq sends a task's registered labels only when it is kicked through its decorated function, so a task kicked by name would run with no `timeout`. The worker's `RegisteredLabelsMiddleware` (`adapters/inbound/jobs/labels.py`) merges the registered labels under each message's own before the task runs, so the registered timeout applies however a task was kicked.
- **Acknowledgement.** A message is acknowledged once its task has finished, whether it returned or raised (Taskiq's `when_saved`, with no result backend). The job row, not the message, decides whether the job is retried.
- **Stream trimming.** taskiq-redis acknowledges with `XACK` and never deletes, so the broker trims the stream approximately to `TALLY_BROKER_STREAM_MAXLEN` on each kick. Trimming can drop an unacknowledged message; its job stays `queued` or `running`, and the sweeper re-dispatches it from the row.
- **Inbound events from AWS.** Scheduled jobs that read queued events, such as SES bounce and complaint notices from SQS, call a use case that reads them through a port.

## Client contract in the code

The [client contract](../client-contract.md) sets the rules for the web and mobile clients. This is where each rule lives.

| Rule | Where | How |
| --- | --- | --- |
| Operation IDs | `adapters/inbound/http/operation_ids.py` | Each router is created with one tag: its domain (`tags=["logbook"]`), or for shared routes its capability (`tags=["jobs"]`). Endpoint functions are named by verb (`list_flights`). `operation_id`, which `create_app` passes to `FastAPI(generate_unique_id_function=...)`, builds `logbook_list_flights` from the route's one tag and its endpoint function's name, never a `name=` override, which only `url_for` should see. Tags are strings (a `StrEnum` member is one). FastAPI also calls it for routes outside the schema, for routes without exactly one tag and for plain `Enum` tags, so it never raises: those get FastAPI's default ID, and the schema check reports any that are in the schema. Webhook and health routes use `include_in_schema=False`. |
| Schema export | `scripts/export_openapi.py` | Builds the app with `Settings.for_schema_export()`, runs `check_schema` from `operation_ids.py`, and writes `openapi.json` with sorted keys, two-space indents, LF line endings (pinned in `.gitattributes`) and a trailing newline. `check_schema` reads the generated schema and fails, naming each route, on a route without exactly one tag, with an operation ID that doesn't start with its tag, or with an operation ID another route shares. `make openapi.fix` writes the file, and `make openapi.check`, part of `make check` and CI, fails when the committed file differs from the export. CI then runs oasdiff against every supported client release's schema. |
| Client versions | `adapters/inbound/http/client_version.py` | Middleware reads `X-Client-Platform` and `X-Client-Version` and compares them with a `ClientVersionPolicy` (minimum version per platform) that bootstrap builds from settings. It returns the `400` `client_version_unsupported` response itself, because exceptions raised in middleware don't reach FastAPI's exception handlers. Webhook routes are exempt. |
| Idempotency | `application/shared/idempotency.py`; `uow.idempotency`; table `idempotency_keys` | See [Idempotency](#idempotency) below. |
| Concurrent edits | The use case; the entry's head row | The use case locks the head row (`SELECT … FOR UPDATE`), compares `base_version` with the current version, and raises `version_conflict` if they differ. |
| Job status | `application/shared/get_job_status.py`, route `GET /jobs/{id}` (tag `jobs`) | Returns status and, when done, a presigned download URL for any file the job produced. |
| Error shape | `domain/shared/errors.py`, `application/shared/errors.py`, `adapters/inbound/http/errors.py` | **Errors carry codes.** `DomainError` and `ApplicationError` both derive from `CodedError` in `domain/shared/errors.py`, which checks at class definition that each concrete error sets its own string `code` (a `TypeError` otherwise), so no two errors share one by inheritance. A base that only groups other errors is declared with `abstract=True` and needs none. Errors that aren't about the domain (authentication, entitlement, idempotency reuse, rate limits, unsupported client) live in `application/shared/errors.py`, also with codes. Every `CodedError` takes one message, which is sent to clients as written, so it holds no internal detail and no other pilot's data.<br>**Use-case errors map by exact class.** The HTTP adapter maps each error to a status in `ERROR_STATUSES`, keyed by exact class rather than through inheritance, so every concrete error needs its own entry. A test imports every module, finds each concrete error, and fails on one without an entry. An unmapped error is re-raised and becomes a `500`.<br>**One body.** Every response is the shared body `{code, message, details}`, built only by `error_response`.<br>**Framework errors.** The adapter replaces FastAPI's handlers for `RequestValidationError` (`422 validation_failed`, with each error's location, message and type but never its input) and for Starlette's `HTTPException`: `HTTP_ERROR_CODES` names the 4xx codes the framework raises (`not_found`, `method_not_allowed`), any other 4xx is `http_error`, and any 5xx is `internal_error` with a fixed message.<br>**Unhandled errors.** Any other exception is caught by `UnhandledErrorMiddleware` (see CORS below), which returns `500 internal_error` with no exception text. It logs the exception's type, message and stack trace with the request's method and path, which can hold personal data until telemetry scrubbing is added. No middleware outside it sees the exception, so exception telemetry is recorded there.<br>**Schema.** `create_app` documents every route's 422 with the shared body, which keeps FastAPI's `HTTPValidationError` schema out of `openapi.json`, and routers declare their other error responses with `responses=error_responses(404, ...)`. |
| CORS | `bootstrap/app.py` | Starlette's `CORSMiddleware`. `create_app` passes its user middleware to `FastAPI(middleware=[...])` as one list, outermost first, and a test checks the order:<br>1. Uvicorn's proxy-headers wrapper, which never responds and must set the client address before anything reads it.<br>2. `CORSMiddleware`, the outermost middleware that can produce a response, so its headers reach every response to an allowed origin, rejections included.<br>3. `UnhandledErrorMiddleware` from `adapters/inbound/http/errors.py`. Starlette always wraps the whole list in its own `ServerErrorMiddleware`, whose 500 responses carry no CORS headers, so unhandled errors are caught inside CORS instead.<br>4. Later middleware, such as the client-version check.<br>Origins come from `TALLY_CORS_ORIGINS`, each exactly as a browser sends it: a lowercase `scheme://host[:port]` without a default port, with an IP address in a browser's form (`[::1]`, `127.0.0.1`), because Starlette compares origins as exact strings; `*` is rejected. The allowed methods are `GET`, `POST`, `PUT`, `PATCH`, `DELETE` and `OPTIONS`, and the allowed request headers are those listed in the client contract, plus the CORS-safelisted `Accept`, `Accept-Language` and `Content-Language`, which Starlette always adds. Credentials aren't allowed, because clients authenticate with bearer tokens, not cookies. A rejected preflight keeps Starlette's plain-text `400`, the one exception to the shared error body, because browsers never expose a preflight response to the page. |

### Idempotency

1. **The router passes the key in.** Write routers take the key as a required `Header(alias="Idempotency-Key")` parameter, so it appears in `openapi.json`. They put it in the command, along with a hash of the request body.
2. **The use case wraps its change.** Inside its unit of work, it calls `run_once(uow, operation, key, request_hash, action)`.
3. **`run_once` inserts the key row first.** It runs `INSERT … ON CONFLICT DO NOTHING RETURNING` on (pilot, operation, key).
   - **A row comes back.** This is the first request. The action runs, and its result (status and body, or a reference for large results) is written to the key row before commit.
   - **No row comes back.** A duplicate exists. Under READ COMMITTED, the insert first waits for a concurrent first request to finish. `run_once` then reads the stored row and returns its result. If the hash differs, it raises `idempotency_key_reused` instead.
4. **A failed action stores nothing.** If the action raises, the transaction rolls back and takes the key row with it, so a retry runs fresh.
5. **Old keys are cleaned up.** A scheduled job deletes keys older than 24 hours.

## Migrations and reference data

- **Table models** subclass `Base` in `adapters/outbound/postgres/tables/`, one module per table. `Base.metadata` carries `NAMING_CONVENTION`, so constraint names are stable: `pk_<table>`, `fk_<table>_<columns>_<referred table>`, `uq_<table>_<columns>`, `ix_<table>_<columns>` and `ck_<table>_<name>`. A check constraint must be given a name, which becomes its suffix.
- **Alembic's `env.py`** targets `load_metadata()`, which imports every module in `tables/` before it returns `Base.metadata`, so a table module nothing else imports can't make autogenerate drop its table. How migrations are deployed is in [tech-stack.md](tech-stack.md#operations).
- **Two login roles.** `tally_aloft_owner` owns the database and, through `pg_database_owner` on Postgres 15 and later, its `public` schema; it runs every migration. `tally_aloft_app` is what the API and workers connect as. Infra, the Compose init script (`compose/postgres-init.sql`) and the test fixture only create the two roles and make the owner the database's owner. They grant nothing else.
- **Grants live in migrations.** Alembic, connected as the owner from `MigrationSettings`, makes every grant, so all environments share one versioned source. The first revision closes `CREATE` on `public` and `TEMPORARY` on the database to `PUBLIC`, gives `tally_aloft_app` `USAGE` on `public`, and sets default privileges `FOR ROLE tally_aloft_owner` so that every new table gives it `SELECT` and `INSERT` only. It fails unless it runs as the owner, and it then checks that the application role can create nothing (no `CREATE` on the database or `public`, no `TEMPORARY`, and none of `SUPERUSER`, `CREATEDB`, `CREATEROLE` or `BYPASSRLS`), failing otherwise, because a `REVOKE` without the grant option only warns.
- **Mutable tables say so.** A revision that creates a table the application updates or deletes from grants `UPDATE` or `DELETE` on it explicitly, so an audit table is append-only by default. A `serial` column or a `DEFAULT nextval()` also needs `USAGE` on its sequence; identity columns and `uuidv7()` keys don't.
- **Timeouts.** `env.py` sends `lock_timeout` (from `TALLY_MIGRATION_LOCK_TIMEOUT`) and `statement_timeout = 0` as session settings when it connects, the equivalent of `SET` and never `SET LOCAL`. The lock timeout therefore holds inside autocommit blocks too, and no database or role default can cut a long migration off. The application's statement timeout never applies to migrations.
- **New tables** build their indexes in the revision that creates them, inside its transaction: an empty table locks nothing anyone waits on. `CONCURRENTLY` is for tables that hold data.
- **Concurrent index builds** go in their own revision, inside `with op.get_context().autocommit_block():`, with `postgresql_concurrently=True`. The block commits the transaction before it, so `env.py` runs each revision in its own transaction (`transaction_per_migration`), as Alembic recommends. A concurrent build that fails, on the lock timeout for example, leaves an `INVALID` index behind. So the block first reads `pg_index.indisvalid` for the index's name: it runs `DROP INDEX CONCURRENTLY` and builds again only if the index is `INVALID`, builds it if it is missing, and keeps a valid one rather than rebuilding it, such as one an operator built ahead of the deploy. Only the name is checked, so an index built ahead of a deploy must match the revision's definition exactly. The check runs in Python, through `op.get_bind()`, because `DROP INDEX CONCURRENTLY` can't run inside a `DO` block. The backend README shows the pattern.
- **Revision files** are named `<date>_<revision>_<slug>.py` and written from `migrations/script.py.mako`. Ruff fixes and formats each new one as Alembic writes it (post-write hooks in `alembic.ini`). `env.py` has no offline (`--sql`) mode.
- Bulk external datasets, such as airport or aircraft registries, load through a bulk-writer port, not row by row through repositories.

## Where things go

| Adding… | Goes in |
| --- | --- |
| A business rule or calculation | `domain/<domain>/` |
| A new domain | Packages in `domain/` and `application/`, a level in both domain contracts (see Domains) |
| A new operation a client can perform | `application/<domain>/<verb>.py`, then add it to `UseCases` |
| A repository or query service | Its Protocol in `application/<domain>/ports.py`; the Postgres implementation in `adapters/outbound/postgres/`; a fake in `tests/fakes/` |
| A list endpoint | A query service returning `Page[T]`, not a repository |
| An endpoint | `adapters/inbound/http/<domain>.py`, calling a use case through a `UseCasesDep` parameter |
| A write endpoint | Also takes the `Idempotency-Key` header; the use case wraps its change in `run_once`. An update also takes `base_version`. |
| A background or scheduled job | A `JobKind` constant and a body use case on `UseCases`; the use case records a job row and calls `dispatch_safely` after its commit; `job_task(kind, body)` added to `TASKS` in `bootstrap/container.py` |
| A new external service | A port in `application/ports/`, an adapter in `adapters/outbound/<technology>/`, wiring in `bootstrap/container.py`, and a fake in `tests/fakes/`. Add a suite in `tests/contracts/` when the fake holds state. |
| A webhook from a vendor | A receiver in `adapters/inbound/http/webhooks/` that verifies the signature, deduplicates on the event ID and calls a use case |
| A database table or column | `adapters/outbound/postgres/tables/`, then an expand-then-contract Alembic migration |
| A setting | `bootstrap/settings.py`, passed to the adapter that needs it |
| A new error a client can act on | A domain or application error with a new `code`, mapped by its exact class in `ERROR_STATUSES` in `adapters/inbound/http/errors.py`. A test fails while it has no entry. |
| A change to an existing endpoint | Add, don't change (see Breaking changes in the [client contract](../client-contract.md)) |

## Testing

| Suite | Tests | Uses |
| --- | --- | --- |
| `unit/domain` | Rules and calculations | Plain pytest; no fixtures needed |
| `unit/application` | Use cases, including authorization, with a denied case for each use case | Fakes from `tests/fakes/` |
| `contracts` | That each stateful fake (the cache, the jobs repositories through their unit of work, object store, job queue) behaves like its real adapter | One suite per port, parametrized over the fake and the real adapter |
| `integration` | Postgres and Valkey adapters, migrations, and the concurrency guarantees: <br>• two concurrent requests with one key produce one write <br>• a duplicate or late delivery doesn't run a job twice <br>• expired leases are recovered <br>• the sweeper re-dispatches a job that was never dispatched <br>• the application role can't update or delete audit rows | testcontainers: one Postgres container per test worker, started on first use, whose roles come from Compose's own init script. Migrations run once per worker, as the owner role; tests connect as the application role. Test-only tables (`tests/support/tables.py`) are created as the owner after the migrations, so the default privileges apply to them. The `database` fixture empties every table but `alembic_version` after each test, through its own owner connection, so tests may commit and use several connections. Valkey likewise runs one container per test worker, started on first use, on Compose's version; the `valkey` fixture deletes every key after each test. Both containers start through `started()` in `tests/support/containers.py`, which retries a start that fails with "address already in use": Docker picks each published host port from the ephemeral range without seeing ports other sockets hold, which on a busy CI runner occasionally collides. Tests that change the schema or the migration state run on a database of their own, where test migrations, written into the test's temporary directory and chained on the real head, run through the real `env.py`. Readiness is tested here too, through the real lifespan and its real checks. Tests that need Docker are marked `container`. |
| `api` | Routing, auth, error mapping, schemas, client-version rejection, idempotent replay, version conflicts, and the API process started on the host | The full app, with outbound adapters replaced by fakes through `overrides` |
| `image` | The release image: its user, size and contents, proxy trust, and shutdown on `SIGTERM` | Docker, marked `container`. Tests run the image's own default command and swap only the app path for `tests.support.probe_app:app`, with `tests/` mounted read-only, so test-only routes reach the real server flags while the image ships no tests. The one exception is the exit-on-`SIGTERM` test, which also shortens `--timeout-graceful-shutdown` to 2 seconds so it doesn't wait out the 25-second drain; the default-command test pins the real value. Worker processes run the same way, from the release image with `tests/` mounted, as `taskiq worker tests.support.worker:broker` with the image contract's flags, reaching the test Postgres and Valkey through `host.docker.internal`; they prove redelivery after `SIGKILL`, both outcomes of `SIGTERM`, and that a bad start exits non-zero. The case that outlasts the 25-second wait shortens `--wait-tasks-timeout` to 1 second. CI passes its built image in `BACKEND_TEST_IMAGE`. |
| `tooling` | The repository's configuration: the Ruff, Pyrefly and aislop policies, the Import Linter contracts, the lockfiles, the `Dockerfile`, `compose.yaml`, the `Makefile`, the CI workflow and the settings list | Plain pytest over the files, plus the tools themselves on probe inputs |

`tests/fakes/__init__.py` assigns each fake to a variable typed as its Protocol, so Pyrefly fails when a fake drifts from its port. The fake repositories sit on a `FakeDatabase`, which holds every fake table and a `Clock` standing in for the database's `now()`. A `FakeUnitOfWork` copies its tables when entered and, on a clean exit, writes back only the rows it changed, so overlapping units of work keep each other's commits; on an exception it drops them. It models no row locks, so concurrency is tested on Postgres. Vendor adapters get opt-in tests against sandbox accounts, outside the default pull-request run.

**Test jobs.** No production job kind exists until a feature needs one, so test-only kinds, bodies and use cases in `tests/support/jobs.py` exercise the job machinery, and `tests/support/worker.py` is the test worker: `create_worker` with the test tasks, the same function `worker.py` calls. Integration tests run it in the test process with Taskiq's own `Receiver`, the image suite runs it in containers, and `compose.e2e.yaml` runs it in Compose, built from the `Dockerfile`'s `dev` stage, the release image plus `tests/`, which is never shipped. It and the modules it imports use only the runtime packages, so they import in both images.

## Enforced boundaries

These Import Linter contracts in `.importlinter` run in CI. A pull request that breaks one fails. The two domain-level contracts list the domains that exist today and grow as domains are added. The file is INI: each contract is an `importlinter:contract:<id>` section, and a list has one item per indented line.

```ini
[importlinter]
root_package = tally_aloft
include_external_packages = true

[importlinter:contract:layers]
name = Dependencies point inward
type = layers
layers =
    tally_aloft.main | tally_aloft.worker
    tally_aloft.bootstrap
    tally_aloft.adapters
    tally_aloft.application
    tally_aloft.domain

# Levels inside each package. Every domain has a package in both.
# Same-level domains go on one line, separated by " | ".
[importlinter:contract:domain-levels-within]
name = Domain levels within domain/ and application/
type = layers
containers =
    tally_aloft.domain
    tally_aloft.application
layers =
    logbook | accounts
    shared
exhaustive = true
exhaustive_ignores =
    ports
    registry

# Levels across the two packages, so a use case can't reach up into a
# higher domain's rules. Each line pairs a domain with its use cases;
# same-level domains join the same line with " : ".
[importlinter:contract:domain-levels-across]
name = Domain levels across domain/ and application/
type = layers
layers =
    tally_aloft.application.logbook : tally_aloft.domain.logbook : tally_aloft.application.accounts : tally_aloft.domain.accounts
    tally_aloft.application.shared : tally_aloft.application.ports : tally_aloft.domain.shared

[importlinter:contract:adapters-independent]
name = Adapters are independent
type = independence
modules =
    tally_aloft.adapters.inbound.*
    tally_aloft.adapters.outbound.*

[importlinter:contract:no-vendor-code]
name = Domain and application have no framework or vendor code
type = forbidden
source_modules =
    tally_aloft.domain
    tally_aloft.application
forbidden_modules =
    fastapi
    starlette
    uvicorn
    pydantic
    pydantic_settings
    sqlalchemy
    asyncpg
    alembic
    redis
    taskiq
    taskiq_redis
    aiohttp
    aiobotocore
    pydantic_ai
    typst
    cryptography
    rfc8785
    jwt
    logfire
    opentelemetry
```

How the two level contracts divide the work:

- **Within each package**, same-level domains are fully independent. The contract is exhaustive, so a domain package missing from it fails CI.
- **Across packages**, a use case can't reach up into a higher domain's rules. Import Linter can't mix independent (`|`) and connected (`:`) modules on one line, so this contract can't stop a use case from reading a same-level domain's types; review covers that rule. The contract also has no containers, so it can't be exhaustive: check that each new domain was added to it.
- **Ports sit on the bottom line** of the cross-package contract, so a port module that imports a domain's rules fails CI. `registry.py` isn't listed, and nothing in a level imports it.

When a new vendor library is adopted, add it to the forbidden list. With `include_external_packages = true`, Import Linter checks forbidden external modules that are present in the import graph; a listed package that is absent from the graph does not itself fail the lint. The list in `.importlinter` is the target list, including transitive packages such as `starlette`, `aiohttp` and `opentelemetry`.

## Conventions

- Package root is `src/tally_aloft`; tests import it as an installed package.
- Use cases are `async`; domain code is synchronous.
- Commands and results are frozen dataclasses; Pydantic stays in adapters and settings.
- Use cases raise domain or application errors. Only `adapters/inbound/http` turns them into HTTP responses.
- Logging inside `domain` and `application` uses the standard `logging` module; Logfire instrumentation is configured in bootstrap and on adapters.
