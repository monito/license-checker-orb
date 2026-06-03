# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [5.0.1]

### Fixed

- The "Prune node_modules to the production closure" step no longer fails on Yarn Berry
  repositories that are passed (or auto-detected) as `pkg-manager: yarn`. Previously the
  classic-`yarn` path always ran `yarn install --production --frozen-lockfile`, and both
  flags are removed and fatal on Yarn Berry (`YN0050`), aborting the build. The prune step
  now detects the actual Yarn major version at runtime and uses
  `yarn workspaces focus --all --production` on Yarn Berry (>= 2) while keeping the legacy
  flags for Yarn Classic (1.x), so both `yarn` and `yarn-berry` work on any Yarn version.

## [5.0.0]

### Changed

- **BREAKING:** The production report is now built by pruning `node_modules` to the
  production closure with the project's package manager and then scanning it, instead
  of using `license-checker --production`. This fixes effectively-empty production
  reports in Yarn/npm/pnpm **workspaces monorepos**, where production dependencies
  declared in workspace packages were previously misclassified as development.
- **BREAKING:** The second CSV artifact is renamed from `<report-name>-development.csv`
  to `<report-name>-all.csv` and now contains every installed dependency (production +
  development). Update any tooling that consumes the report by its previous path.
- The forbidden-license gate (when not `production-only`) now runs against the
  all-dependencies set.

### Added

- `pnpm` is now a supported `pkg-manager` value (in addition to `npm`, `yarn`, `yarn-berry`).
- `pkg-manager` parameter on the `validate` and `validate-copyleft` commands, selecting
  how `node_modules` is pruned to the production closure.

### Notes

- Yarn Berry projects must use `nodeLinker: node-modules` in `.yarnrc.yml` (required for
  `license-checker` to find `node_modules`).
- Classic `yarn` v1 uses `yarn install --production`; this is accurate for single-package
  repos and best-effort for v1 workspaces (the all-dependencies gate remains the safety net).

## [4.0.0]

### Changed

- **BREAKING:** Development dependencies are now checked and reported by default. Previously only production dependencies were validated. Set `production-only: true` to restore the previous behavior.
- **BREAKING:** The production CSV report artifact is now named `<report-name>-production.csv` (was `<report-name>.csv`). Update any tooling that consumes the report by its previous path.
- Update license
- Upgrade circleci/node version

### Added

- A separate CSV report is now generated for development dependencies: `<report-name>-development.csv`.
- `production-only` option (false by default) to restrict the checks and reports to production dependencies.
- `exclude-private` option (true by default) to exclude private packages from the checks and reports.

## [3.0.2]

### Changed

- Improve README.md

## [3.0.1]

### Changed

- Update `orb-tools` and CI

## [3.0.0]

### Changed

- Remove global install step

### Breaking change

- Removed install command

## [2.0.1] - 2022-04-26

### Changed

- Updated Node.js version to 14.17.1

## [2.0.0] - 2022-04-07

### Added

- new `license-checker/validate` command
- new `license-checker/validate-copyleft` command

### Breaking change

- Remove `forbidden` parameter from `license-checker/validate-copyleft` job. Use `license-checker/validate` job if you want to customize the list of forbidden licenses.

### Changed

- Update default node executor version to use the `lts` tag
- Upgrade `circleci/node` orb
- Upload license report artifacts relative to `app-dir`

## [1.2.0] - 2021-06-04

### Added

- Support to override the CI install command used by the Node orb to install packages

## [1.1.0] - 2021-05-12

### Added

- 1 new job `validate-copyleft` to check licenses against a predefined list of Copyleft licences

### Changed

- Documentation improvement
- Some renaming in test files

## [1.0.0] - 2021-05-11

### Added

- Initial Release:
- 1 job `validate`
- 2 commands (`install`, `execute`)
- 1 default executor
