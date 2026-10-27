Technical · Supply chain · October 2026

# Tally Aloft — Dependency Supply Chain

The rules every repository follows for the third-party code it installs or runs: packages and GitHub Actions. They exist to keep a compromised release, such as a malicious version published from a hijacked maintainer account, out of the lockfiles and the CI runs.

## Minimum release age: 7 days

- **The rule.** Dependency resolution only takes a package release that was published at least 7 days earlier. This applies to direct and transitive dependencies, runtime and development alike, in every repository. A malicious release is usually found and yanked within days, so waiting a week means it never reaches a lockfile.
- **The package manager enforces it,** not review:

  | Repository | Setting | Exception for one package |
  | --- | --- | --- |
  | Backend | `exclude-newer = "7 days"` in `pyproject.toml`'s `[tool.uv]`. `uv.lock` records the span (`exclude-newer-span = "P7D"`), not a date, so the lock stays valid as days pass and `--locked` installs don't fail. A tooling test checks both. | `exclude-newer-package = { <name> = "<RFC 3339 timestamp>" }` |
  | Backend npm tools | `minimumReleaseAge: 10080` (minutes) in `pnpm-workspace.yaml`, for the tools that ship only as npm packages, such as aislop. A tooling test checks it. | `minimumReleaseAgeExclude` |
  | Clients (proposed) | `minimumReleaseAge: 10080` (minutes) in `pnpm-workspace.yaml`, which needs pnpm 10.16 or later | `minimumReleaseAgeExclude` |

- **Lower bounds.** A manifest's lower bound never names a release younger than 7 days, because resolution would fail. `make packages.update` (see [make commands](make-commands.md)) upgrades only as far as the rule allows.
- **Exceptions** are only for a security fix that can't wait a week. Exempt that one package, as narrowly as the package manager allows: in uv, a cutoff timestamp just after the fixed release rather than `false`, which would drop the rule for that package entirely. Put the reason and the date the exception can go (7 days after the release) on its line, and remove it on that date.
- **Installs never resolve.** CI and images install exactly what the committed lockfile holds (`uv sync --locked`; `pnpm install --frozen-lockfile`), so the rule only has to hold when the lockfile changes. Running a tool doesn't resolve either: by default `pnpm exec` installs and rewrites `pnpm-lock.yaml` when `node_modules` is out of date, so the backend sets `verifyDepsBeforeRun: error` in `pnpm-workspace.yaml` to fail instead. A tooling test checks it.
- **Packages, not launchers.** A tool is installed from the package that contains it, so the lockfile covers its dependencies. A package that downloads the tool when it runs, such as the PyPI `aislop` package, which runs `npx aislop@<version>`, bypasses the lockfile and the 7-day rule for everything it downloads.

## GitHub Actions

- **Commit SHAs, with the release in a comment.** Every third-party action is pinned to the full 40-character commit SHA of an exact release, followed by that release's tag in a comment, such as `actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1`. Never a tag or a branch alone. A tooling test checks this form in each workflow. Chosen over exact release tags, because whoever controls an action's repository can move a tag to new code, as the March 2025 compromise of `tj-actions/changed-files` did with existing version tags, and a commit SHA can't be moved. The comment keeps the version readable at a glance, so an update still shows what it changes.
- **Updating a pin.** Resolve the release tag to its commit with `git ls-remote https://github.com/<owner>/<repo> refs/tags/<tag> 'refs/tags/<tag>^{}'`. When the tag is annotated, the `^{}` line is the commit; pin that, not the tag object's SHA. Update the SHA and the comment together.
- **7 days old.** A new action version is adopted only once its release is at least 7 days old, as for packages. No tool enforces this, so check the release's published date when you update a pin.
