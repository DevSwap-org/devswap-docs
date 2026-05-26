# Contracts

Solidity `=0.8.24`, evm `shanghai`, OpenZeppelin v5.1.0 (vendored in
`contracts/lib/`). Two contracts: `DevSwapToken` and `DevSwapEscrow`.

## DevSwapToken (`src/DevSwapToken.sol`)

ERC-20 platform token **$DSWP**: capped at **100,000,000**, burnable,
`Ownable2Step`. Burned by the escrow's buyback path via `burn()` (never sent to
`address(0)` — OZ's ERC20 reverts on a zero-address transfer).

## DevSwapEscrow (`src/DevSwapEscrow.sol`)

USDT escrow with the full task lifecycle and the **Option-C** inline
buyback-burn. Inherits `ReentrancyGuard`, `Pausable`, `Ownable2Step`; uses
`SafeERC20`. Assumes a **non-fee-on-transfer** USDT (canonical BSC USDT, 18 dec).

### Constants

| Name                 | Value     | Meaning                                   |
|----------------------|-----------|-------------------------------------------|
| `FEE_BPS`            | 150       | 1.5% owner fee                            |
| `BUYBACK_BPS`        | 150       | 1.5% buyback-and-burn                     |
| `BPS_DENOMINATOR`    | 10_000    | basis-point denominator                   |
| `MIN_SUBMIT_TIMEOUT` | 1 day     | floor for `setSubmitTimeout`              |
| `MAX_SUBMIT_TIMEOUT` | 60 days   | ceiling for `setSubmitTimeout`            |
| `MAX_SLIPPAGE_BPS`   | 1_000     | 10% ceiling for the inline-buyback guard  |

Defaults set in the constructor: `submitTimeout = 14 days`,
`autoBuybackEnabled = true`, `buybackSlippageBps = 300` (3%).

### Status state machine

```
None ─createTask─▶ Open ─acceptTask─▶ Accepted ─submitTask─▶ Submitted
                    │                    │                       │
                    │ cancelTask         │ cancelTask(timeout)   │ releaseFunds
                    ▼                    ▼                       ▼
                 Cancelled           Cancelled                Released
   Accepted/Submitted ─raiseDispute─▶ Disputed ─resolveDispute─▶ Released | Cancelled
```

### Lifecycle functions

| Function | Caller | Effect |
|---|---|---|
| `createTask(amount, metadataHash) → taskId` | anyone | escrows `amount` USDT (needs prior approve), status → Open |
| `acceptTask(taskId)` | non-client | status → Accepted, stamps `acceptedAt` |
| `submitTask(taskId, deliveryHash)` | developer | status → Submitted |
| `releaseFunds(taskId)` | client | status → Released, runs `_payout` (Option C) |
| `cancelTask(taskId)` | client | refund if Open, or Accepted past `submitTimeout`. **Not** `whenNotPaused` — clients can always reclaim |
| `raiseDispute(taskId)` | client or developer | Accepted/Submitted → Disputed |
| `resolveDispute(taskId, payDeveloper)` | owner | Disputed → Released (split) or Cancelled (refund) |
| `executeBuybackBurn(minDswpOut, deadline)` | owner or keeper | swap `buybackReserve` USDT → $DSWP, burn; CEI zeroes reserve first |

### The split (`_payout`)

On release: **97% developer + 1.5% fee** are transferred **first** (payment is
never market-dependent). Then the **1.5% buyback**:

- if `autoBuybackEnabled`: `try this.autoBuybackAndBurn(buyback)` — quote via
  `getAmountsOut`, `minOut` from `buybackSlippageBps`, swap on PancakeSwap, burn.
- on any failure, **or** if disabled: accrue to `buybackReserve`, emit
  `BuybackDeferred` for a later bulk `executeBuybackBurn`.

`developerNet = amount − fee − buyback`, so the three parts always sum to
`amount` (no dust left behind).

### Admin / safety

`setFeeRecipient`, `setKeeper` (address(0) disables keeper), `setSubmitTimeout`
(bounded), `setAutoBuybackEnabled`, `setBuybackSlippageBps` (≤ `MAX_SLIPPAGE_BPS`),
`pause`/`unpause`. All `onlyOwner`; ownership becomes multisig + timelock before
mainnet (P5).

### Events

`TaskCreated`, `TaskAccepted`, `TaskSubmitted`, `FundsReleased`,
`TaskCancelled`, `DisputeRaised`, `DisputeResolved`, `BuybackBurned`,
`BuybackDeferred`, plus admin-update events. These are the subgraph's input.

### Custom errors

`ZeroAddress`, `ZeroAmount`, `InvalidTaskStatus`, `NotClient`, `NotDeveloper`,
`NotParty`, `NotAuthorized`, `ClientCannotAcceptOwnTask`, `CannotCancel`,
`NothingToBuyback`, `InvalidTimeout`, `InvalidSlippage`, `OnlySelf`.

## Quality gates (run from `contracts/`)

```bash
forge build --sizes
forge test -vvv
forge coverage --report summary          # Escrow ≥ 95%, Token ≥ 90% (currently 100%)
FOUNDRY_PROFILE=ci forge test --fuzz-runs 10000
forge test --match-test invariant_
slither . --filter-paths "lib|test|script"
forge fmt --check
```

## Deployment addresses (BSC testnet 97 — keys to be rotated)

| Contract | Address |
|---|---|
| Escrow | `0xCEE07220dEC813f8A58b7Da73349dabbc4005840` |
| DSWP | `0x2DD2Cd306f32cd6709d4316EF0df125235654734` |
| mock USDT (test) | `0xE950eb93aCa1f29848f5cBac61d78657e3c97287` |

BSC hard facts: USDT `0x55d398326f99059fF775485246999027B3197955` (18 dec),
PancakeSwap V2 router `0x10ED43C718714eb63d5aA57B78B54704E256024E`.
