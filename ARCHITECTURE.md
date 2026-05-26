# Architecture

DevSwap is a P2P decentralized marketplace for software services on **BNB Smart
Chain (BSC)**, settled in **USDT** escrow, with a platform utility token
**$DSWP** reduced via a separated **buyback-and-burn** mechanism.

> Source-of-truth order on any conflict: `STATE.md` > `PLAN.md` > `TASKS.md` >
> this file > prior knowledge.

## System diagram (logical)

```
                ┌────────────────────────────────────────────┐
   Browser ───▶ │  web/  (Next.js 14 App Router, wagmi/viem)  │
                │  Explore · Post · Task · Wallet · Admin      │
                └───────┬─────────────────────┬───────────────┘
                        │ reads/writes          │ reads (indexed)
                        ▼                       ▼
            ┌────────────────────┐    ┌────────────────────────┐
            │ BSC (chain 56/97)  │    │ subgraph/ (The Graph)   │
            │  DevSwapEscrow     │───▶│  Task/User/Buyback/...   │
            │  DevSwapToken      │    └────────────────────────┘
            │  USDT · Pancake V2 │
            └─────────┬──────────┘
                      │ events
        ┌─────────────┼───────────────────────────┐
        ▼             ▼                            ▼
  keeper/        notifications/             supabase/ (off-chain
  (buyback cron) (FCM push)                  mirror + search, blocked
                                             on keys)
```

## Folders (flat layout — see ADR-0001)

| Folder           | Purpose                                                        | Toolchain            |
|------------------|---------------------------------------------------------------|----------------------|
| `contracts/`     | `DevSwapEscrow`, `DevSwapToken`, tests, deploy scripts          | Foundry, OZ v5.1.0   |
| `web/`           | dApp (Vercel root dir)                                          | Next 14, wagmi v2    |
| `subgraph/`      | The Graph indexer                                              | graph-cli, AS/wasm   |
| `keeper/`        | buyback cron                                                   | Node, viem           |
| `notifications/` | FCM HTTP v1 sender                                             | Node (zero-dep)      |
| `supabase/`      | off-chain mirror schema + RLS                                 | Postgres / Supabase  |
| `cloudflare/`    | edge config (SSL, WAF, rate-limit, redirects) via API          | bash + CF API        |
| `docs/`          | architecture, contracts, security, UX, i18n, ADRs              | Markdown             |
| `prompts/`       | role playbooks for repeatable tasks                            | Markdown             |

## On-chain ↔ off-chain boundary

- **On-chain (authoritative):** escrow balances, task state machine, fund
  splits, buyback-burn, ownership. Never trust off-chain state for money.
- **Indexed (read-optimized):** the subgraph projects events into
  `Task`/`User`/`BuybackBurn`/`GlobalStats` for fast list/detail/stat reads.
- **Off-chain (convenience only):** Supabase mirror for search and
  token↔address mapping for per-user push. Treated as a cache; the chain wins.

## Trust model

- Funds custody is the escrow contract only. The frontend, indexer, keeper, and
  Supabase are **non-custodial helpers** — compromising any of them must not move
  funds.
- The keeper can only trigger the already-permissioned buyback path; it cannot
  redirect payouts.
- Admin (owner) actions are limited to dispute resolution + safety toggles
  (pause, auto-buyback enable, slippage), all guarded by `Ownable2Step`.

## Data flow: task lifecycle

1. Client posts a task → USDT pulled into escrow (`TaskCreated`).
2. Developer accepts (`TaskAccepted`) → submits delivery (`TaskSubmitted`).
3. Client releases (or timeout) → `FundsReleased`: 97% dev, 1.5% fee, 1.5%
   buyback (burned inline or deferred). Disputes route through the owner panel.
4. Subgraph indexes each event; web reads list/detail/stats from it; push
   notifications fire from indexed events (per-user path blocked on Supabase).

See `docs/CONTRACTS.md` for the contract-level detail and `docs/SECURITY.md` for
the threat model.
