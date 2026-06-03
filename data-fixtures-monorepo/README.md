# Monorepo test fixture (yarn-berry workspaces)

Used by the orb's integration tests to verify monorepo-correct production
reporting. The root is `private` with **no `dependencies`** and only a
`devDependencies` entry (`chalk`); the real production dependency (`ms`, MIT)
lives in the `apps/app-a` workspace, mirroring a typical workspaces monorepo.

`.yarnrc.yml` sets `nodeLinker: node-modules` because `license-checker` scans
`node_modules` (Yarn Berry's default PnP linker has none).

Expectations after the orb runs:
- `*-production.csv` contains `ms` (the workspace app's prod dep) — not just the root.
- `*-all.csv` contains both `ms` and `chalk`.
- A forbidden-license gate over the production closure that forbids `MIT` fails.
