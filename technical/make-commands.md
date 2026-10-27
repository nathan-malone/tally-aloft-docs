Technical · Make commands · October 2026

# Tally Aloft — Make Commands

Each code repository has a `Makefile` at its root that holds every command a developer or CI runs: install, lint, type-check, test, build and run. The README, `CLAUDE.md` and the CI workflow call make targets, never the tools directly, so a command changes in one place. This document sets the conventions every repository's `Makefile` follows and the targets they share, so `make check` means the same thing in the backend and in the clients.

*Decided:* every code repository uses a `Makefile` this way, and the backend's is built. *Proposed:* the clients' commands in the [shared targets](#shared-targets) table, confirmed when the clients repository's tooling is set up. `tally-aloft-infra` follows the same rules if it needs a `Makefile`.

Make was chosen over listing raw commands, which had to be kept the same by hand in the README, `CLAUDE.md`, the CI workflow and the tooling tests. It was chosen over just or Task because GNU Make is one standard package on every platform: Linux distributions and GitHub's runners have it, macOS has it with the Xcode Command Line Tools, and Windows needs `winget install ezwinports.make`.

## Rules

- **Commands aren't echoed.** The `Makefile` declares `.SILENT:`, so only the tools' output shows. `make -n <target>` prints the commands without running them.
- **Labels name the job, not the tool.** Target names, help text, CI job and step names, and the README's and `CLAUDE.md`'s descriptions of targets say what is checked or done ("Check types", "Run tests"), not which tool does it. Swapping a tool then changes only the recipe. Tools are named where the text is about the tool itself, such as its configuration rules, and in this document's tool columns.
- **`make` alone prints help.** `help` is the default goal. It lists each target with one line, written out with `$(info …)`, because the usual `grep` and `awk` help generators don't run under `cmd.exe`. Each target has its own `.PHONY` line above it, and a tooling test checks that every target is phony and in the help.
- **Recipes run under `cmd.exe` and `sh`.** GNU Make on Windows runs recipes with `cmd.exe` when no `sh.exe` is on `PATH`, which is the usual case from PowerShell. So each recipe line is one plain command: no pipes, redirects, `&&`, `;`, globs, environment-variable syntax (`$VAR`, `%VAR%`), `^` or single quotes. Double quotes are fine. Logic goes in make functions such as `$(if …)`, or in a script that the repository's own toolchain runs (`uv run python scripts/…`, `pnpm exec tsx scripts/…`). A tooling test checks recipes for this syntax.
- **GNU Make 3.81 or later.** macOS ships 3.81, so the `Makefile` doesn't use `.ONESHELL`, `!=`, `::=`, `.RECIPEPREFIX` or grouped targets (`&:`).
- **`check` holds every check.** `check` lists every `<noun>.check` target as a prerequisite, then `test`. `check.fast` lists the same checks, then `test.fast`. CI runs every prerequisite of `check` as its own step. A new check is added in those three places, and tooling tests fail if one is missed. Prerequisites, rather than calls to `$(MAKE)`, let `make -k check` run every check and report every failure. Without `-k`, make stops at the first failure.
- **One at a time.** `.NOTPARALLEL:` keeps the checks serial and in order even when `-j` is in `MAKEFLAGS`, so their output doesn't interleave.
- **Check targets print a banner.** The recipe of each target that `check` or `check.fast` runs starts with `$(banner)`, defined as `$(info ==> $@)`, so the output of `make check` shows which check is running.
- **Only the lock targets change the lockfile.** `packages.lock` and `packages.update` may rewrite it. Every other target installs or runs from the lockfile as it is and fails if it's stale (`uv run --locked`, `pnpm install --frozen-lockfile`, and `pnpm exec` under `verifyDepsBeforeRun: error`), as CI does. So `make check` never silently re-locks after a `pyproject.toml` edit.
- **Variables come from the command line.** A target that takes input reads a variable set on the command line, as in `make test ARGS="-k settings"`. The `Makefile` assigns each one a default with `=`, not `?=`, so a same-named environment variable doesn't leak in. Windows machines often set `PLATFORM`, for example. The shared names are `ARGS` (extra arguments for the underlying tool) and `PACKAGE` (one package to act on).
- **CI calls make targets only.** Every `run:` step in the CI workflow is a `make` target, and the workflow's tooling test checks that. Actions that install the toolchain itself, such as `setup-uv`, and checks that ship as an action, such as `oasdiff-action`, stay actions.
- **Images don't use make.** A `Dockerfile` keeps its own build steps, such as `uv sync --locked --no-dev`, because they build the release image rather than a developer environment. The release image never contains make.
- **Tests that run make clear its environment.** A tooling test that runs make removes `MAKEFLAGS`, `MFLAGS`, `MAKELEVEL` and `MAKEOVERRIDES` from the child's environment. Otherwise, under `make test`, the outer make's variables, such as `ARGS`, reach the inner one.

## Naming

A target is `<noun>.<verb>` or `<noun>.<qualifier>.<verb>`: lowercase words joined by dots, naming what is acted on and then the action. Examples are `lint.check`, `image.build` and `packages.outdated.show`. The verbs are:

| Verb | Meaning |
| --- | --- |
| `check` | Reads and fails on a problem, changing nothing. CI runs these. |
| `fix` | Changes files to fix what the matching `check` reports. |
| `install`, `lock`, `update` | Dependencies: install exactly what is locked, update the lockfile after a manifest change, upgrade locked versions. |
| `show` | Prints information and changes nothing. |
| `build` | Builds an artifact, such as an image or a bundle. |
| `run` | Runs one process in the foreground on the host. |
| `up`, `down` | Start and remove containers. |

`help`, `test` and `check` are single words because they are used most. `.fast` marks a quicker variant of `test` or `check` that skips the slow or Docker-dependent part.

## Shared targets

Every code repository has these targets, with the same meaning. The tool columns name what each target runs; the exact commands and flags live in each repository's `Makefile`. The clients column is proposed, mapped onto the [client monorepo's](clients/structure.md#monorepo-rules) Turborepo tasks.

| Target | Meaning | Backend | Clients (proposed) |
| --- | --- | --- | --- |
| `packages.install` | Install the locked dependencies, failing if the lockfile is stale. Never rewrites the lockfile. | `uv sync --locked`, then `pnpm install --frozen-lockfile` for the npm tools | `pnpm install --frozen-lockfile` |
| `packages.lock` | Update the lockfile after editing the manifest by hand. | `uv lock`, then `pnpm install --lockfile-only` | `pnpm install --lockfile-only` |
| `packages.update` | Upgrade every locked package within the manifest's ranges and the [7-day minimum release age](supply-chain.md), or only `PACKAGE`, then install. | `uv lock --upgrade` and `pnpm update`, then install. Each package manager ignores a `PACKAGE` it doesn't manage. | `pnpm update --recursive` |
| `packages.outdated.show` | Show which direct dependencies have newer releases. | `uv tree --outdated`, which lists every direct dependency and marks the outdated ones, then `pnpm outdated` | `pnpm outdated --recursive`, which lists only the outdated ones |
| `lint.check` | Linter and formatter, check only. | Ruff | Biome (the `//#lint` task) |
| `lint.fix` | Apply the linter's safe fixes and format, then report what is left. | Ruff | Biome (`biome check --write`) |
| `types.check` | Type checker. | Pyrefly | TypeScript and `astro check` (the `typecheck` task) |
| `boundaries.check` | Module-boundary rules. | Import Linter | Steiger (the `steiger` task) |
| `slop.check` | Patterns AI coding agents leave behind, such as narrative comments, swallowed exceptions and stub code. Fails on any finding. | aislop | aislop, once over the whole repository |
| `test` | The full test suite that CI runs on each pull request. `ARGS` goes to the runner. | pytest, including the `container` tests | Vitest and Jest (the `test` task) |
| `test.fast` | The tests that need no Docker or other slow setup. | pytest without the `container` tests | Same as `test` until there is a slow part |
| `check` | Every `*.check` target, then `test`: what must pass before a commit. | `lint.check`, `types.check`, `boundaries.check`, `slop.check`, `test` | The same, plus `codegen.check` |
| `check.fast` | `check` with `test.fast` in place of `test`. | | |

## Repository targets

Targets that only one repository needs follow the same naming.

- **Backend.** `image.build` builds the image, for a `PLATFORM` when one is given (CI passes `linux/arm64`). `api.run` runs the API on the host with reload, loading `.env` (see [Local development](backend/tech-stack.md#developer-tooling)). `containers.up` and `containers.down` start and remove the Compose `app` profile. Later stories add their processes and tools here, such as a `worker.run`, a `migrations.apply` or an `openapi.check`, rather than documenting raw commands.
- **Clients (proposed).** `codegen.check` for the `codegen:check` task, an end-to-end target for the Playwright run in CI, and a `run` target for each app, named when the repository is set up.
