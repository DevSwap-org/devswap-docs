# Smart Contracts

Solidity `=0.8.34`, evm `shanghai`, OpenZeppelin v5 (vendored under `contracts/lib/`). The on-chain stack: the platform token `DevSwapToken`, the escrow series (currently `DevSwapEscrowV2_6`), and the staked arbiter pool `DevSwapArbiterPool`.

## DevSwapToken (`src/DevSwapToken.sol`)

ERC-20 platform token **`$DSWP`** — capped at **100,000,000**, burnable, `Ownable2Step`. Burned by the escrow's buyback path via `burn()` — never sent to `address(0)` (OZ ERC-20 reverts on a zero-address transfer).

## DevSwapEscrowV2_6 (`src/DevSwapEscrowV2_6.sol`)

USDT escrow with the full task lifecycle, the **Option-C** inline buyback-burn, and the **symmetric 3 % dispute bond** with a **4-way split** on resolution. Inherits `ReentrancyGuard`, `Pausable`, `Ownable2Step`; uses `SafeERC20`. Assumes a **non-fee-on-transfer** USDT (canonical BSC USDT, 18 decimals).

### Constants

| Name | Value | Meaning |
|------|------|---------|
| `FEE_BPS` | 150 | 1.5 % platform fee |
| `BUYBACK_BPS` | 150 | 1.5 % buyback-and-burn |
| `BPS_DENOMINATOR` | 10_000 | basis-point denominator |
| `DISPUTE_BOND_BPS` | 300 | symmetric 3 % dispute bond |
| `MIN_SUBMIT_TIMEOUT` | 1 day | floor for `setSubmitTimeout` |
| `MAX_SUBMIT_TIMEOUT` | 60 days | ceiling for `setSubmitTimeout` |
| `MAX_SLIPPAGE_BPS` | 1_000 | 10 % ceiling for the inline-buyback guard |
| `VOTING_WINDOW` | 7 days | dispute-panel voting window |

Defaults set in the constructor: `submitTimeout = 14 days`, `autoBuybackEnabled = true`, `buybackSlippageBps = 300` (3 %).

### Status state machine

```
None ─createTask─▶ Open ─acceptTask─▶ Accepted ─submitTask─▶ Submitted
                    │                    │                       │
                    │ cancelTask         │ cancelTask (timeout)  │ releaseFunds
                    ▼                    ▼                       ▼
                 Cancelled           Cancelled                Released

Accepted / Submitted ─raiseDispute─▶ Disputed ─voteOnDispute (× 3)─▶ Voting
                                          │                            │
                                          │ timeoutDispute              │ finalizeDispute
                                          ▼                            ▼
                                  Cancelled (refund)             Released (4-way split)
```

### Lifecycle functions

| Function | Caller | Effect |
|---|---|---|
| `createTask(amount, metadataHash) → taskId` | anyone | escrows `amount` USDT (needs prior `approve`), status → Open |
| `acceptTask(taskId)` | non-client | status → Accepted, stamps `acceptedAt` |
| `submitTask(taskId, deliveryHash)` | developer | status → Submitted |
| `releaseFunds(taskId)` | client | status → Released, runs `_payout` (Option C) |
| `cancelTask(taskId)` | client | refund if Open, or Accepted past `submitTimeout`. **Not** `whenNotPaused` — clients can always reclaim |
| `raiseDispute(taskId)` | client or developer | Accepted / Submitted → Disputed; **posts the 3 % symmetric bond**; `requestPanel` draws 3 unique arbiters from `DevSwapArbiterPool` |
| `voteOnDispute(taskId, voteForDeveloper)` | panel member | records a vote; 2 / 3 majority auto-finalises |
| `finalizeDispute(taskId)` | anyone | post-window resolution; runs the **4-way split** (50 / 35 / 10 / 5) |
| `claimWinnerDeposit(taskId)` / `claimArbiterReward(taskId, idx)` | winner / majority arbiter | pull-payment of the resolved share |
| `executeBuybackBurn(minDswpOut, deadline)` | owner or keeper | swap `buybackReserve` USDT → `$DSWP`, burn; CEI zeroes reserve first |

### The settlement split (`_payout`)

On a normal release: **97 % developer + 1.5 % platform fee** are transferred **first** (payment is never market-dependent). Then the **1.5 % buyback**:

- If `autoBuybackEnabled`: `try this.autoBuybackAndBurn(buyback)` — quote via `getAmountsOut`, `minOut` from `buybackSlippageBps`, swap on PancakeSwap, burn.
- On any failure, **or** if disabled: accrue to `buybackReserve`, emit `BuybackDeferred` for a later bulk `executeBuybackBurn`.

`developerNet = amount − fee − buyback`, so the three parts always sum to `amount` (no dust left behind).

### The dispute split (`_finalize`)

When a dispute resolves, the **combined 6 % bond** (3 % from each party) is split:

| Recipient | Share of bond |
|-----------|--------------:|
| Winner (refund of own bond + share of loser's bond) | 50 % |
| Majority panel (split equally among the 2 / 3 majority) | 35 % |
| Buyback-burn (PancakeSwap V2 → `burn()`) | 10 % |
| Platform fee bucket | 5 % |

The disputed milestone amount itself goes to the winner per the panel's ruling.

### Admin / safety

`setFeeRecipient`, `setKeeper` (`address(0)` disables the keeper), `setSubmitTimeout` (bounded), `setAutoBuybackEnabled`, `setBuybackSlippageBps` (≤ `MAX_SLIPPAGE_BPS`), `setArbiterPool` (plug-in upgrade path), `pause` / `unpause`. All `onlyOwner`; ownership becomes multisig + timelock before mainnet (see [`SECURITY-AUDIT.md §1`](SECURITY-AUDIT.md)).

### Events

`TaskCreated`, `TaskAccepted`, `TaskSubmitted`, `FundsReleased`, `TaskCancelled`, `DisputeRaised`, `DisputePanelStored`, `VoteCast`, `DisputeFinalized`, `BuybackBurned`, `BuybackDeferred`, plus admin-update events. These are the subgraph's input.

### Custom errors

`ZeroAddress`, `ZeroAmount`, `InvalidTaskStatus`, `NotClient`, `NotDeveloper`, `NotParty`, `NotAuthorized`, `ClientCannotAcceptOwnTask`, `CannotCancel`, `NothingToBuyback`, `InvalidTimeout`, `InvalidSlippage`, `OnlySelf`, `UseVotingFlow`, `NoClaim`.

## Quality gates (run from `contracts/`)

```bash
forge build --sizes
forge test -vvv
forge coverage --report summary          # Escrow ≥ 95 %, Token ≥ 90 %
FOUNDRY_PROFILE=ci forge test --fuzz-runs 10000
forge test --match-test invariant_
slither . --filter-paths "lib|test|script"
forge fmt --check
```

## Deployment addresses (BSC testnet, chainId 97)

| Contract | Address |
|---|---|
| `DevSwapEscrowV2_6` (current) | `0x22633bd98d6F9AD4dF499b77429459F5574B4dFe` |
| `DevSwapArbiterPool` | `0x747A7a306F12Fce896F08e9A62a7ef83f1d53C95` |
| `$DSWP` (testnet) | `0x2DD2Cd306f32cd6709d4316EF0df125235654734` |
| Mock USDT (testnet) | `0xf24e2651A0A63EAf99A3dcE3F3Fb4ff997A8c3F7` |

BSC hard facts: real USDT `0x55d398326f99059fF775485246999027B3197955` (18 decimals), PancakeSwap V2 router (mainnet) `0x10ED43C718714eb63d5aA57B78B54704E256024E`, PancakeSwap V2 router (testnet) `0xD99D1c33F9fC3444f8101754aBC46c52416550D1`.

## Earlier versions (historical)

Earlier escrow versions remain deployed on testnet for audit traceability:

| Version | Address | Status |
|---------|---------|--------|
| `DevSwapEscrow` (V1) | `0xCEE07220dEC813f8A58b7Da73349dabbc4005840` | Superseded |
| `DevSwapEscrowV2_1` | `0x67Eca35d3d23401d53Fba988759F8A649BA67c3e` | Superseded |
| `DevSwapEscrowV2_4` | `0xa1aF0da1494Db38924fC2055B9deA79B8b376F47` | Superseded by V2.6 |

The mainnet `$DSWP` token contract is `0x52a68C09f3237B4CB0944F58Ed1CA110a49bE1d9` (deployed for the future buyback-burn mechanism; no active offering — see [`whitepaper.md §5`](whitepaper.md)).
