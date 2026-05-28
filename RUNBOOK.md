# DevSwap — Operations Runbook

Operational procedures for running the DevSwap contracts and dApp safely. Pair this with [`SECURITY-AUDIT.md`](SECURITY-AUDIT.md) (threat model) and [`adr/ADR-0003-arbiter-hardening.md`](adr/ADR-0003-arbiter-hardening.md) (dispute design).

> Status: **testnet**. Mainnet operations additionally require the gates in [`SECURITY-AUDIT.md §1`](SECURITY-AUDIT.md). The keys in use today are testnet-only and will be rotated as part of the mainnet handover. **Never paste a private key into a shell that logs, a file, chat, or a PR.**

## 0. Addresses & roles (BSC testnet, chainId 97)

| Component | Address |
|---|---|
| `DevSwapEscrowV2_6` (current) | `0x22633bd98d6F9AD4dF499b77429459F5574B4dFe` |
| `DevSwapArbiterPool` | `0x747A7a306F12Fce896F08e9A62a7ef83f1d53C95` |
| `$DSWP` token (testnet) | `0x2DD2Cd306f32cd6709d4316EF0df125235654734` |
| Mock USDT (testnet) | `0xf24e2651A0A63EAf99A3dcE3F3Fb4ff997A8c3F7` |
| PancakeSwap V2 router (testnet) | `0xD99D1c33F9fC3444f8101754aBC46c52416550D1` |

Earlier escrow versions (V1 / V2.1 / V2.2 / V2.4) remain deployed for historical audit traceability but are not the active contract for new jobs.

**Roles.** `owner` (admin: pause, fee / keeper / timeout config, arbiter-pool parameters — **cannot** resolve disputes itself in V2.4+); `arbiter` (staked panel member; resolves disputes); `keeper` (triggers `executeBuybackBurn` and milestone auto-release); `client` / `developer` (marketplace users).

`RPC=https://data-seed-prebsc-1-s1.binance.org:8545` for the `cast` examples below. Read-only calls are safe at any time. State-changing calls require the matching role-key.

## 1. Arbiter operating rules (V2.4+)

**Golden rule: keep ≥ 3 active arbiters at all times.** This guarantees that removing one never strands an open dispute (see [`ADR-0003 §4`](adr/ADR-0003-arbiter-hardening.md) — the all-arbiters-removed freeze is an intentional but operationally-prevented edge case).

- **Eligibility.** An arbiter can vote on a dispute only if it is active when the panel is drawn (`requestPanel`). Arbiters added after the panel was drawn are not eligible for it (anti-puppet). Removed arbiters lose all voting power immediately.
- **Add an arbiter.** Stake via `stake(amount)` on `DevSwapArbiterPool`; the address becomes immediately eligible to be drawn into new panels.
- **Remove an arbiter.** `requestUnstake()` starts a 30-day cooldown; `withdraw()` afterwards. Withdrawal is blocked while `openDisputeCount > 0` so an arbiter cannot exit mid-dispute.
- **Rotation.** Add the replacement first, confirm `activeArbiterCount ≥ 3`, then start the outgoing arbiter's unstake.
- **Quarterly review.** Inspect the arbiter set; confirm contact / on-call coverage; flag inactive members.

## 2. Dispute resolution SOP

1. Either party calls `raiseDispute(jobId, index)` and posts the **symmetric 3 % bond**; the milestone is frozen, `disputeRaisedAt` is snapshotted, and `requestPanel` draws 3 unique arbiters from the pool.
2. Each panel member calls `voteOnDispute(jobId, index, voteForDeveloper)` within the 7-day voting window. Two matching votes (2 / 3 majority) auto-finalize.
3. Anyone may call `finalizeDispute(jobId, index)` after the window closes; the **4-way split** runs:
   - 50 % to the winner (refund of own bond + share of loser's bond)
   - 35 % to the majority panel (split equally among the 2 / 3 majority)
   - 10 % to buyback-burn (PancakeSwap V2 → `burn()`)
   - 5 % to the platform fee bucket
4. Winners and majority arbiters call `claimWinnerDeposit` / `claimArbiterReward` to receive their share (pull-payment, strict CEI, `nonReentrant`).
5. Tie (no majority): resolution defaults to the client (conservative, see [`ADR-0013`](adr/ADR-0013-arbiter-staking.md)).

## 3. Emergency procedures

### 3.1 Owner-key compromise (highest severity)
1. **`pause()`** immediately — blocks new jobs / accepts / submits / releases / disputes. Note that `cancelMilestone` stays available so clients can always reclaim funds (intentional escape hatch).
2. From a safe device, `transferOwnership(newSafeOwner)` then `acceptOwnership()` (`Ownable2Step`). Prefer a multisig as the new owner.
3. Rotate `feeRecipient` / `keeper` if those keys shared exposure.
4. `unpause()` only after ownership is secured. Post-incident: publish a blameless postmortem.

### 3.2 Arbiter-key compromise
1. The compromised arbiter cannot exit while `openDisputeCount > 0`. If they have not voted, the panel will rely on the other two members (2 / 3 majority).
2. If the compromised arbiter has voted maliciously, dispute history must be reviewed and any related claims paused.
3. Once safe, the arbiter unstakes via the normal cooldown.

### 3.3 Pause / unpause
- `pause()` / `unpause()` are `onlyOwner`. Use pause for any active incident, a bad deploy, or an observed exploit attempt. Communicate status to users (banner + announcement) when paused.

### 3.4 Buyback / market issues (illiquid pool, MEV)
- Inline buyback **never blocks payment** — it defers the 1.5 % to `buybackReserve` on any failure.
- If sandwiching becomes material: `setAutoBuybackEnabled(false)` → the 1.5 % accrues to the reserve; drain it later with `executeBuybackBurn(minDswpOut, deadline)` using an off-chain–computed `minDswpOut` (never `0` on the bulk path — it is sandwich-exposed).
- Tune `setBuybackSlippageBps(bps)` (≤ `MAX_SLIPPAGE_BPS` = 1000 / 10 %) for the inline path.

### 3.5 Funds-stuck / unresolvable dispute
- A dispute with no eligible arbiter freezes (funds stay escrowed — safe). Resolution: ensure at least one arbiter eligible at that dispute's panel-draw block exists, or wait for the 30-day timeout fallback (`timeoutDispute` returns funds to the client conservatively).

## 4. Weekly operations dashboard

| Metric | Source | Healthy |
|---|---|---|
| Active arbiter count | `activeArbiterCount` | **≥ 3** |
| Open disputes & age | Subgraph / `getMilestone` status = Disputed | Resolved < 7 d |
| `buybackReserve` (USDT) | `cast call … "buybackReserve()(uint256)"` | Drained periodically |
| Total `$DSWP` burned | Subgraph `GlobalStats.totalDswpBurned` / `/status` | Trending up |
| Deployer / keeper BNB balance | `cast balance` | Enough for gas |
| `paused` state | `cast call … "paused()(bool)"` | `false` (unless incident) |
| Owner address | `cast call … "owner()(address)"` | Expected multisig / role-key |
| `$DSWP` / USDT LP depth (mainnet) | DEX | Sufficient before enabling buyback |
| Web app health | `curl -sf https://devswap.pro/en` | `200` |
| CI status | GitHub Actions | Green on `main` |

## 5. Escalation paths

1. **L1 — on-call operator.** Triages alerts, runs read-only checks, calls `pause()` if a live exploit is suspected (when holding the owner key; otherwise escalate to L2).
2. **L2 — owner / multisig signer(s).** Ownership actions (pause, transfer, arbiter pool parameters, config). Pre-mainnet this is a 3-of-5 multisig + timelock.
3. **L3 — security lead / auditor.** Exploit analysis, postmortem, fix-and-redeploy decision.
4. **External.** The independent auditor (PeckShield / CertiK) for confirmed vulnerabilities; legal counsel for user-fund or regulatory impact.

**Severity → action:** funds-at-risk → L1 pause + page L2 / L3 now · degraded (buyback / MEV) → L2 within 24 h · cosmetic → normal PR flow.

## 6. Routine operations

- **Buyback burn.** Keeper or owner runs `executeBuybackBurn(minDswpOut, deadline)` when `buybackReserve > 0` and LP depth is adequate; `minDswpOut` computed off-chain from `getAmountsOut`.
- **Milestone auto-release.** After `reviewTimeout` (7 d), the developer or keeper may `claimMilestone(jobId, index)` for an unresponsive client.
- **Deploy / verify a new contract.** `forge script script/DeployV2_6.s.sol --rpc-url … --broadcast`, then `forge verify-contract … --chain 97 --etherscan-api-key …` (key from env, never inline). Update the `ESCROW_V2_6` + `NEXT_PUBLIC_ESCROW_V2_6_*` env vars afterwards.
- **Pre-mainnet.** Audit clean → owner → multisig + timelock → LP locked → deploy on chain 56 → smoke test. See [`SECURITY-AUDIT.md §1`](SECURITY-AUDIT.md) and [`§11`](SECURITY-AUDIT.md).
