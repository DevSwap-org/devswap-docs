# Security

This project custodies third-party funds. The conservative option always wins.
See also the audit notes in `SECURITY-AUDIT.md`.

## Principles

- **CEI everywhere.** Status/state is written before any external call on every
  fund-moving path.
- **ReentrancyGuard** on every external mutating fund path
  (`createTask`, `releaseFunds`, `cancelTask`, `resolveDispute`,
  `executeBuybackBurn`).
- **SafeERC20** for all token movement; `forceApprove` before swaps.
- **Ownable2Step** so ownership transfer is two-phase (no fat-finger handover).
- **Pausable** for incident response — but `cancelTask` is intentionally **not**
  gated by `whenNotPaused` so clients can always reclaim funds.
- **Burn via `burn()`**, never a transfer to `address(0)` (OZ ERC20 reverts on
  zero-address).

## Payment-safety property (Option C)

The single most important invariant: **a developer's payment is never blocked by
the market.** `_payout` transfers the 97% + 1.5% fee first, then attempts the
1.5% buyback inside a `try/catch` self-call. Any swap failure (illiquid pool,
slippage, deadline) is caught and the 1.5% is deferred to `buybackReserve`. The
developer is already paid before the swap is ever attempted.

`autoBuybackAndBurn` is `OnlySelf` (rejects external callers) and deliberately
**not** `nonReentrant`: it runs inside the caller's guarded scope, and any
genuine reentry into a guarded function reverts and is absorbed by the
surrounding `try/catch`.

## Invariants (enforced by tests)

- **Solvency:** escrow USDT balance ≥ Σ(amount of non-terminal tasks) +
  `buybackReserve`. Verified by Foundry invariant tests (16k calls).
- **Conservation:** `developerNet + fee + buyback == amount` for every release.
- **No double-spend:** terminal statuses (Released/Cancelled) are sticky; every
  mutator re-checks status.
- **Slippage bound:** inline buyback `minOut` derives from on-chain
  `getAmountsOut` × (1 − `buybackSlippageBps`); `executeBuybackBurn` takes an
  explicit `minDswpOut` + `deadline`.

## Threat model (selected)

| Threat | Mitigation |
|---|---|
| Reentrancy on payout/buyback | CEI + ReentrancyGuard + self-call try/catch isolation |
| Malicious/illiquid swap blocking pay | pay-first; buyback deferral on failure |
| MEV sandwich on buyback | slippage guard + deadline; batched bulk burn option |
| Fee-on-transfer USDT accounting drift | documented assumption: canonical non-FoT USDT only |
| Owner key compromise | P5 gate: multisig 3-of-5 + timelock before mainnet |
| Frontend/indexer/keeper compromise | non-custodial by design; cannot redirect funds |

## Test coverage

**400 + tests** across 19 Foundry suites: unit, fuzz @ 10 000 runs, invariant @ 16 384 calls, reentrancy, plus mainnet-fork tests against the real PancakeSwap router (auto-skip without `BSC_RPC_URL`). Escrow line coverage ≥ 95 %; function coverage 100 %. Slither: 0 high / 0 medium.

## Secrets policy

- **No secrets in the client.** Every secret lives server-side only. Only `NEXT_PUBLIC_*` (non-secret) values reach the browser bundle.
- `.env*` files are gitignored and must never be committed.
- Testnet-only keys are isolated from any mainnet path and will be rotated as part of the mainnet handover.
- Never print secret values in logs, chat, or commits.

## Pre-mainnet gates

Independent audit (PeckShield / CertiK) → multisig + timelock ownership → LP lock → qualified-counsel review. None of these may be skipped. The full gate list lives in [`SECURITY-AUDIT.md §1`](SECURITY-AUDIT.md).

## Reporting

Report vulnerabilities privately to the security contact in [`SECURITY.md`](https://github.com/DevSwap-org/.github/blob/main/SECURITY.md) — **do not** open a public issue for an exploitable finding.
