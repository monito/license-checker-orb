# License Checker Orb [![CircleCI Build Status](https://circleci.com/gh/monito/license-checker-orb.svg?style=shield "CircleCI Build Status")](https://circleci.com/gh/monito/license-checker-orb) [![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/monito/license-checker-orb/main/LICENSE)

A Circle CI orb to check the licenses used by your project — for both the production dependency closure and all installed dependencies, including in workspaces monorepos — based on [license-checker](https://www.npmjs.com/package/license-checker)

## Usage

For full usage guidelines, see the [orb registry listing](https://circleci.com/developer/orbs/orb/monito/license-checker).

## Monorepo & package manager support

The orb supports `npm`, `yarn` (classic v1), `yarn-berry`, and `pnpm` via the
`pkg-manager` parameter, and produces correct production reports for both
single-package repos and **workspaces monorepos**.

Rather than trusting `license-checker --production` (which only understands the
root `package.json` and reports an effectively empty production set in a workspaces
monorepo), the orb prunes `node_modules` to the production closure using the
package manager itself, then scans what remains:

| `pkg-manager` | production prune |
|---|---|
| `npm` | `npm ci --omit=dev` |
| `yarn` | `yarn install --production --frozen-lockfile` |
| `yarn-berry` | `yarn workspaces focus --all --production` |
| `pnpm` | `pnpm install --prod --frozen-lockfile` |

Two CSV artifacts are produced:

- `<report-name>-production.csv` — the production dependency closure only.
- `<report-name>-all.csv` — every installed dependency (production + development).

By default the forbidden-license gate runs against the all-dependencies set
(a superset of production). Set `production-only: true` to gate and report on the
production closure alone.

> **Yarn Berry note:** `license-checker` scans `node_modules`, which Yarn Berry's
> default Plug'n'Play linker does not create. Yarn-berry projects must set
> `nodeLinker: node-modules` in `.yarnrc.yml`.

## Releasing

Publishing a new version of the orb is **driven by git tags**. The CI is split into two pipelines (the [Orb Development Kit](https://circleci.com/docs/orb-author/) layout):

- [`.circleci/config.yml`](.circleci/config.yml) — the *setup* pipeline. On every commit it lints and packs the orb source, then triggers the `test-deploy` pipeline.
- [`.circleci/test-deploy.yml`](.circleci/test-deploy.yml) — runs the integration tests on every commit, and **publishes to the registry only when a release tag is pushed**.

A production version is published when a tag matching `^v[0-9]+\.[0-9]+\.[0-9]+$` (e.g. `v4.0.0`) is pushed. The `orb-tools/publish` job (gated on that tag and on all tests passing) publishes `monito/license-checker@X.Y.Z` to the registry — the orb version is the tag without the leading `v`.

### Cutting a release

1. Choose the next [semantic version](https://semver.org/) for the change (major / minor / patch).
2. Update [`CHANGELOG.md`](CHANGELOG.md) with the new version and merge it to `main`.
3. From `main`, create and push the matching tag:

   ```bash
   git checkout main && git pull
   git tag v4.0.0
   git push origin v4.0.0
   ```

4. CircleCI validates the orb (lint, pack, integration tests) and, on success, publishes the new version to the registry.

> Publishing requires an organization-owner CircleCI API token, supplied to the `orb-tools/publish` job via the `orb-publishing` [context](https://circleci.com/docs/contexts/).

## Contributing

We welcome [issues](https://github.com/monito/license-checker-orb/issues) to and [pull requests](https://github.com/monito/license-checker-orb/pulls) against this repository!

For further questions/comments about this orb, visit the [discussions](https://github.com/monito/license-checker-orb/discussions).

For more information about the Orb publishing process, see:
- https://circleci.com/developer/orbs/orb/circleci/orb-tools
- https://circleci.com/docs/creating-orbs/