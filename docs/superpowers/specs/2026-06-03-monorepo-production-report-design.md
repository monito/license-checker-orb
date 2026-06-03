# Monorepo-correct production license reporting

**Date:** 2026-06-03
**Status:** Approved design (pending spec review)
**Repo:** `monito/license-checker` orb

## Problem

`license-checker` classifies production vs development relative to the **root**
`package.json` only — it walks the root's `dependencies` vs `devDependencies`.
It has no understanding of Yarn/npm/pnpm **workspaces**.

In a typical workspaces monorepo the root `package.json` is `private`, declares
`workspaces` (e.g. `["apps/*"]`), has only repo-level `devDependencies`, and has
**no `dependencies`**. The real production dependencies live in each workspace's
`package.json` and are hoisted into the root `node_modules`. Because they are not
referenced by the root `package.json`, `license-checker` treats them as
development/extraneous.

Measured in a real consuming repo (`@gifsa/main-data`, Yarn 4.9.1 workspaces):

| Invocation | Packages reported |
|---|---|
| `license-checker --csv` (no flag) | 2143 |
| `license-checker --development --csv` | 2143 (identical to no-flag) |
| `license-checker --production --csv` | 1 (only the root package) |

So the orb's production report is effectively empty for workspaces monorepos, and
the production/development split is meaningless. The current orb
([src/commands/validate.yml](../../../src/commands/validate.yml)) builds the
production report with `license-checker --production`, which is exactly the broken
classification.

## Goal

Make the orb produce a **correct, complete production license report** for any
repository — single-package or workspaces monorepo — across the package managers
the orb supports, without relying on `license-checker`'s `--production` flag.
Keep the copyleft/forbidden gate (`validate-copyleft`) consuming the now-correct
reports. Keep the orb generic and easy to extend to additional build managers.

## Core principle

Do not trust `license-checker --production` in a workspaces world. Instead:

1. **All-dependencies report** = `license-checker --csv` on the **full install**
   (no flag). Universal, always correct and complete — it catches a forbidden
   license *anywhere* in the dependency tree. This is the safety-net gate.
2. **Production report** = reduce `node_modules` to the production closure using
   the package manager's own production-install, then `license-checker --csv`
   (no flag) over what remains.

Because the all-report is simply the full scan and every supported manager has a
production-install command, the mechanism is **uniform**: no `--production` flag,
no workspace detection, no per-repo special-casing. The only thing that varies per
manager is a single prune command.

This replaces the previous, misleadingly-named **development** report with an
**all-dependencies** report. The two reports are now:

- `<report-name>-production.csv` — production closure only.
- `<report-name>-all.csv` — every installed dependency (production + development).

> **Naming change (consumer impact):** the second artifact is renamed from
> `<report-name>-development.csv` to `<report-name>-all.csv`. Consumers that key
> off the old `-development.csv` name (e.g. `@gifsa/main-data`) must update their
> artifact handling. This is an intentional, breaking output change and warrants a
> major version bump of the orb.

## Per-manager prune commands

| `pkg-manager` | prune command |
|---|---|
| `npm` | `npm ci --omit=dev` |
| `yarn` (classic v1) | `yarn install --production --frozen-lockfile` |
| `yarn-berry` | `yarn workspaces focus --all --production` |
| `pnpm` | `pnpm install --prod --frozen-lockfile` |

Notes / verified facts:

- `yarn workspaces focus --all --production` works in **both** single-package and
  workspaces yarn-berry repos — verified locally with Yarn 4.9.1 (a non-workspace
  root exits 0; the root is treated as the sole workspace). No workspace detection
  needed.
- `circleci/node@7`'s `install-packages` already supports `pnpm` (and `bun`) — its
  `pkg-manager` enum is `npm, yarn, yarn-berry, pnpm, bun` — so adding `pnpm` to
  this orb and threading it to the install step works out of the box.
- Yarn Berry defaults to **PnP** (no `node_modules`). `license-checker` requires a
  real `node_modules`, so yarn-berry consumers must use `nodeLinker: node-modules`
  in `.yarnrc.yml`. This is a pre-existing requirement of using `license-checker`
  with yarn-berry; it will be documented and the test fixture sets it.
- yarn v1: the uniform prune is strictly ≥ today's behavior — accurate for
  single-package repos, best-effort for v1 workspaces (whose `--production` pruning
  is historically imperfect), with the all-report gate as the safety net.
- `npm ci --omit=dev` requires npm ≥ 8.3 (the `--omit` flag); satisfied by all
  current `cimg/node` images. Documented as a requirement.
- `bun` is not added now, but the `case` structure is the extension point: adding
  `bun) bun install --production ;;` plus the enum value is the only change needed.

## Affected components

### `src/commands/validate.yml`

Add a `pkg-manager` parameter (enum `npm | yarn | yarn-berry | pnpm`, default
`yarn` to match the job default and keep existing direct-command callers working).
Update the `report-name` description (suffixes are now `-production` and `-all`).
Replace the production-first step sequence with the uniform flow below. The flow
runs in `working_directory: << parameters.app-dir >>`.

Ordering is essential: the **full install must be scanned (and gated) before
pruning**, because pruning destroys the dev dependencies. Reports are written
before any gate can fail, so both CSV artifacts are always produced.

Step sequence:

1. **Configure** — set `LC_EXCLUDE_PRIVATE` (`--excludePrivatePackages` when
   `exclude-private` is true, else empty) into `$BASH_ENV`. (Unchanged from today.)
2. *(when not `production-only`)* **Build all-dependencies report** — `when: always`
   — `npx license-checker $LC_EXCLUDE_PRIVATE --csv > /tmp/<report-name>-all.csv`.
3. *(when not `production-only`)* **All-dependencies license check** — `npx
   license-checker $LC_EXCLUDE_PRIVATE --summary --failOn "<forbidden>"`. Fails the
   job on a forbidden license anywhere in the full install.
4. *(when not `production-only`)* **All-dependencies summary** — `when: on_fail` —
   `npx license-checker $LC_EXCLUDE_PRIVATE --summary`.
5. **Prune to production closure** — `when: always` — `case` on `pkg-manager`
   selecting the prune command from the table above. `when: always` so the
   production report is still produced even if the all-deps gate (step 3) failed.
6. **Build production report** — `when: always` — `npx license-checker
   $LC_EXCLUDE_PRIVATE --csv > /tmp/<report-name>-production.csv` (no
   `--production` flag — everything installed is now production).
7. *(when `production-only`)* **Production license check** — `npx license-checker
   $LC_EXCLUDE_PRIVATE --summary --failOn "<forbidden>"`. Fails the job on a
   forbidden license in the production closure.
8. *(when `production-only`)* **Production summary** — `when: on_fail` — `npx
   license-checker $LC_EXCLUDE_PRIVATE --summary`.
9. **Store production report** — `store_artifacts`, `when: always`, destination
   `<< app-dir >>/<report-name>-production.csv` (unchanged path scheme).
10. *(when not `production-only`)* **Store all-dependencies report** —
    `store_artifacts`, `when: always`, destination
    `<< app-dir >>/<report-name>-all.csv`.

Gate semantics: when not `production-only`, the gate is on the **all** report
(a superset of production), so a forbidden license in any workspace's production
deps fails the build. When `production-only`, the gate is on the production report.
`store_artifacts` and `on_fail` summary steps run via `when: always` / `when:
on_fail`, so they execute even after a gate step fails — preserving today's
"artifacts always produced, summary printed on failure" behavior.

### `src/commands/validate-copyleft.yml`

Add the `pkg-manager` parameter and pass it through to `validate`. No other change
(it already delegates with the copyleft `forbidden` default).

### `src/jobs/validate.yml` and `src/jobs/validate-copyleft.yml`

- Add `pnpm` to the `pkg-manager` enum (now `npm | yarn | yarn-berry | pnpm`).
- Pass `pkg-manager: << parameters.pkg-manager >>` into the `validate` /
  `validate-copyleft` command invocation.

`node/install-packages` already receives `pkg-manager` and supports all four
values, so the full install at job level is unchanged.

### Backward-compatibility caveat

Direct callers of the `validate` / `validate-copyleft` **commands** (not the jobs)
that previously omitted `pkg-manager` now get the default `yarn` prune
(`yarn install --production`). On a non-yarn repo this would be wrong, so such
callers must set `pkg-manager`. The jobs always pass the correct value, so job
users are unaffected. This is documented in the command descriptions and README.

## Testing

### New fixture: `data-fixtures-monorepo/`

A minimal yarn-berry workspaces monorepo:

- Root `package.json`: `private: true`, `packageManager: yarn@4.x`, **no
  `dependencies`**, a `workspaces: ["apps/*"]` field, and a repo-level
  `devDependencies` entry (e.g. a small dev-only tool) to prove dev deps land in
  the all-report but not the production report.
- `.yarnrc.yml` with `nodeLinker: node-modules` (required for `license-checker`).
- `yarn.lock` committed (immutable install in CI).
- `apps/app-a/package.json`: `private: true`, a real **MIT** production dependency
  (a small, stable package, e.g. `ms`) in `dependencies`.

### Integration tests (`.circleci/test-deploy.yml`)

Add a `pkg-manager: yarn-berry` job/run over `data-fixtures-monorepo` that:

1. Installs the fixture (full install), then exercises `license-checker/validate`.
2. **Positive assertion:** `grep`s `/tmp/<report-name>-production.csv` for the
   workspace's production dependency (`ms`) and fails if absent — directly proving
   the regression is fixed (the workspace app's prod dep now appears in the
   production report, where before only the root package would).
3. **All-report assertion:** confirms the dev-only package appears in
   `-all.csv` but **not** in `-production.csv`.
4. **Negative assertion (CI stays green):** runs the prune + `npx license-checker
   --failOn MIT` over the pruned tree inside a bash step with an inverted exit code
   (`! npx license-checker ... --failOn MIT`), asserting a forbidden license in a
   workspace app's production deps now trips the gate. Inverting the exit code lets
   the integration job itself pass while still verifying the failure path.

Existing single-package fixture tests (`data-fixtures`, yarn v1) must continue to
pass under the uniform prune (`yarn install --production --frozen-lockfile` then
plain scan), confirming no regression for single-package repos.

The publish workflow's `requires:` list in `.circleci/test-deploy.yml` is extended
to include the new monorepo integration job.

## Documentation

- Update `validate` / `validate-copyleft` command and job descriptions to describe
  the production vs all reports, the `-production` / `-all` artifact suffixes, and
  the `pkg-manager`-driven prune.
- `README.md`: explain monorepo support, the `nodeLinker: node-modules` requirement
  for yarn-berry, the report rename, and the supported package managers.
- `CHANGELOG.md`: note the breaking output rename (`-development.csv` →
  `-all.csv`), monorepo-correct production reports, and `pnpm` support → major bump.

## Out of scope

- `bun` support (extension point left in the `case`).
- True dev-only-diff report (development = all − production). Rejected in favor of
  the simpler, robust all-report; the gate remains correct either way.
- Changing how the full install happens (`node/install-packages` is unchanged).
