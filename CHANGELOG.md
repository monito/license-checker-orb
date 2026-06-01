# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
