# ADR-0001 — Repository Structure: keep flat, do not migrate to monorepo

- **Status:** Accepted
- **Date:** 2026-05-22
- **Supersedes:** —

## Context

The original project plan assumed a workspace layout (`apps/web`, `packages/contracts`, `packages/subgraph`, …). In practice, the repository was already live with a flat, folder-per-project structure.

The repository already exists and is **live**:

- `web/` is deployed on Vercel with **Root Directory = `web`** and git
  auto-deploy connected to `main`. The custom domain `devswap.pro` resolves
  through Cloudflare → Vercel to this project.
- `contracts/` is a self-contained Foundry project (`foundry.toml`,
  `remappings.txt`, vendored `lib/`) with three CI workflows referencing these
  paths.
- `subgraph/`, `keeper/`, `notifications/`, `supabase/`, `cloudflare/` are
  independent workspaces, each with its own toolchain and lockfile.

There is **no root `package.json`** and no workspace manager. The repo is a
"poly-folder" of independent projects, not an npm/pnpm workspace.

## Decision

**Keep the existing flat structure.** Do not restructure to `apps/*` +
`packages/*`.

Map the original layout onto the existing folders:

| Prompt (assumed)        | This repo (actual)         |
|-------------------------|----------------------------|
| `apps/web`              | `web/`                     |
| `packages/contracts`    | `contracts/`               |
| `packages/subgraph`     | `subgraph/`                |
| `services/keeper`       | `keeper/`                  |
| `services/notifications`| `notifications/`           |
| `infra/*`               | `cloudflare/`, `supabase/` |
| `docs/`                 | `docs/`                    |

## Consequences

**Positive**

- Live Vercel deploy (`rootDirectory=web`) keeps working with zero config churn.
- Foundry paths, remappings, vendored `lib/`, and the three CI workflows
  (`contracts.yml`, `security.yml`, `web.yml`) stay valid.
- The live domain and SSL chain are untouched.
- No risk of a large, untestable "move everything" diff before Phase 1 work.

**Negative / trade-offs**

- No shared `packages/*` for cross-app code reuse (e.g. a shared ABI/types
  package). Mitigation: the subgraph and web each keep their own copy of the
  ABI; the contract address + ABI are the only shared surface and are small.
- Cross-cutting scripts must be run per-folder (each has its own lockfile).

**Revisit when:** a genuine need for shared TypeScript packages appears (e.g. a
shared `@devswap/contracts-types` consumed by both `web/` and `keeper/`). At
that point introduce a pnpm workspace incrementally without moving `web/`.
