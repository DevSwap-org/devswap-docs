# ADR-0008 — Switch Deploy Target: Vercel → Cloudflare Workers (OpenNext)

- **Status:** Accepted
- **Date:** 2026-05-23
- **Relates:** supersedes the Vercel deploy path; runbook `docs/CLOUDFLARE-DEPLOY.md`; workflow
  `.github/workflows/cloudflare-deploy.yml`.

## Context

DevSwap's web app (Next.js 14 App Router, next-intl) deployed to **Vercel** (auto-deploy from `main`,
Root Directory `web`). DNS already runs through **Cloudflare** (zone `devswap.pro`, proxied → Vercel
origin). Two pressures motivated a full move to Cloudflare:

1. **Operational:** the Vercel **free-tier daily build rate-limit** repeatedly blocked live deploys, stalling the live site behind CI-green `main`.
2. **Consolidation:** DNS, WAF, caching, and analytics already live on Cloudflare — running the app runtime on the same platform removes a hop (CF → Vercel origin) and unifies edge + app.

The decision to migrate to Cloudflare was driven by those two pressures. The app is a good fit for **Cloudflare Workers via the OpenNext adapter** (`@opennextjs/cloudflare`): it is SSG + dynamic API routes with **no ISR / revalidate**, so no incremental cache (R2 / KV) is required — `defineCloudflareConfig()` defaults suffice.

## Decision

- **Deploy target is Cloudflare Workers** built by **OpenNext** (`@opennextjs/cloudflare`), routed to
  `devswap.pro/*` (zone `devswap.pro`) per `web/wrangler.jsonc`. `nodejs_compat`, `compatibility_date
  2024-12-30`, assets bound as `ASSETS`, worker entry `.open-next/worker.js`.
- **Build:** `pnpm -C web cf:build` (`next build` → OpenNext workerd bundle). **Verified green** on `main`
  (includes the `/start` hybrid router and all routes; 2 benign vendored-chunk warnings only).
- **CI/CD:** `.github/workflows/cloudflare-deploy.yml` — builds on every push to `main` (verifies
  workerd-buildability), and **deploys only when `CLOUDFLARE_API_TOKEN` is set** (graceful, like the
  indexer/keeper workflows). Manual `workflow_dispatch` supported.
- **Dev bindings:** `next.config.mjs` calls `initOpenNextCloudflareForDev()` (no-op for build/prod).
- **Vercel:** kept as a **dormant rollback** until the Cloudflare deploy is verified live, then its
  auto-deploy is disabled (do not delete the project immediately — it is the rollback path).
- **Cutover** is the Worker route: once the Worker is deployed with route `devswap.pro/*`, Cloudflare
  serves it ahead of the Vercel origin — see the runbook for the exact DNS/route steps and rollback.

## Consequences

- **Build-time public config** (`NEXT_PUBLIC_*`, inlined) must be set as **GitHub repo Variables** for CI
  builds (today they live in Vercel's env); the full list is in the runbook.
- **Server secrets** used by API routes (`GITHUB_CLIENT_SECRET`, `SUPABASE_SERVICE_ROLE_KEY`, `FIREBASE_SERVICE_ACCOUNT_JSON`, `PINATA_JWT`, `NEXTAUTH_SECRET`, …) must be set as **Worker secrets** (`wrangler secret put`) — they are NOT inlined and NOT in the workflow.
- **Credential requirement.** The deploy step needs a `CLOUDFLARE_API_TOKEN` with Workers Scripts + Workers Routes permissions for the account; until that token is provided the workflow builds (verifying workerd-buildability) but does not deploy.
- **Rollback.** Disable / remove the Worker route → Cloudflare falls back to the Vercel origin (kept warm).
- **Process.** The project's deploy convention is the one in this ADR plus the runbook; it overrides any earlier "Vercel-only" assumption for this repository.
