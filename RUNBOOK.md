# DevSwap — Operations Runbook

Operational procedures for running the DevSwap contracts and dApp safely. Pair with
`SECURITY-AUDIT.md` (threat model) and `docs/decisions/ADR-0003-arbiter-hardening.md` (dispute design).

> Status: **testnet**. Mainnet operations additionally require the P5 gate (audit + multisig +
> timelock + LP lock). The keys currently in use are testnet-only and will be rotated by the owner
> after completion. **Never paste a private key into a shell that logs, a file, chat, or a PR.**

## 0. Addresses & roles (BSC testnet, chainId 97)

| Thing | Address |
|---|---|
| DevSwapEscrowV2_1 (current, hardened) | `0x67Eca35d3d23401d53Fba988759F8A649BA67c3e` |
| DevSwapEscrow (V1, legacy/live) | `0xCEE07220dEC813f8A58b7Da73349dabbc4005840` |
| $DSWP token | `0x2DD2Cd306f32cd6709d4316EF0df125235654734` |
| Test USDT | `0xE950eb93aCa1f29848f5cBac61d78657e3c97287` |
| PancakeSwap V2 router (testnet) | `0xD99D1c33F9fC3444f8101754aBC46c52416550D1` |

> The original (non-hardened) V2 `0x13e8…6AC` is **deprecated/abandoned** — do not use.

**Roles:** `owner` (admin: pause, fee/keeper/timeout config, arbiter registry — **cannot** resolve
disputes), `arbiter` (resolves disputes; registered via the timelock), `keeper` (triggers
`executeBuybackBurn` + milestone auto-release), `client`/`developer` (marketplace users).

`RPC=https://data-seed-prebsc-1-s1.binance.org:8545` for the `cast` examples below (read-only calls
are safe to run anytime; state-changing calls need the owner/arbiter key and are the owner's action).

## 1. Arbiter operating rules (V2.1)

**Golden rule: keep ≥3 active arbiters at all times.** This guarantees that removing one never
strands an open dispute (see ADR-0003 §4 — the all-arbiters-removed freeze is an intentional but
operationally-prevented edge case).

- **Eligibility:** an arbiter can resolve a dispute only if `isArbiter[arbiter]` is true **and**
  `arbiterSince[arbiter] <= disputeRaisedAt`. An arbiter added after a dispute opened is **not**
  eligible for it (anti-puppet). Removed arbiters lose all power immediately.
- **Add an arbiter (48h timelock):**
  1. `queueArbiter(addr)` (owner) — starts the 48h clock; visible on-chain.
  2. wait ≥ `ARBITER_TIMELOCK` (48h), then `executeArbiter(addr)` (owner) — registers it.
  3. `cancelArbiterChange(addr)` aborts a pending add at any time before step 2.
  - UI: `/v2/admin` (owner-only). CLI (read state):
    `cast call 0x67Eca… "isArbiter(address)(bool)" <addr> --rpc-url $RPC`
- **Remove an arbiter (immediate):** `removeArbiter(addr)` (owner). Use the moment an arbiter is
  compromised, unresponsive, or rotated out — there is no delay by design.
- **Rotation:** to replace arbiter X with Y: `queueArbiter(Y)` → wait 48h → `executeArbiter(Y)` →
  confirm count ≥3 → `removeArbiter(X)`. Never drop below 3 mid-rotation.
- **Quarterly:** review the arbiter set; remove inactive members; confirm contact/on-call coverage.

## 2. Dispute resolution SOP

1. Either party calls `raiseDispute(jobId, index)` → milestone frozen, `disputeRaisedAt` snapshotted.
2. An **eligible** arbiter reviews the off-chain evidence (IPFS spec + delivery hash).
3. Arbiter calls `resolveDispute(jobId, index, payDeveloper)` via `/v2/job/[id]`:
   - `true` → pay developer (97/1.5/1.5 split, buyback runs); client's `disputesLost++`.
   - `false` → refund client; developer's `disputesLost++`.
4. If no eligible arbiter is available, add one via the 48h timelock (note: a newly-added arbiter is
   **not** eligible for already-open disputes — keep ≥3 so this never blocks resolution).

## 3. Emergency procedures

### 3.1 Owner key compromise (highest severity)
1. **`pause()`** immediately (owner) — blocks new jobs/accepts/submits/releases/disputes. Note
   `cancelMilestone` stays available so clients can always reclaim funds (intentional escape hatch).
2. From a **safe** device, `transferOwnership(newSafeOwner)` then `acceptOwnership()` (Ownable2Step).
   Prefer a multisig as the new owner.
3. `removeArbiter` any arbiter the attacker may have queued/executed; `cancelArbiterChange` any
   pending adds. The 48h timelock means an attacker could not have added an arbiter that is eligible
   for any currently-open dispute.
4. Rotate `feeRecipient`/`keeper` if those keys shared exposure.
5. `unpause()` only after ownership is secured. Post-incident: write a blameless postmortem
   (`engineering:incident-response`).

### 3.2 Arbiter key compromise
1. `removeArbiter(badArbiter)` (owner) — immediate, no timelock.
2. Verify ≥3 arbiters remain; if not, the affected open disputes freeze until a peer (eligible at
   `disputeRaisedAt`) resolves them — this is why the standing set must be ≥3.
3. Investigate any disputes that arbiter resolved while compromised.

### 3.3 Pause / unpause
- `pause()` / `unpause()` are owner-only. Use pause for any active incident, a bad deploy, or an
  observed exploit attempt. Communicate status to users (banner + announcement) when paused.

### 3.4 Buyback / market issues (illiquid pool, MEV)
- Inline buyback **never blocks payment** — it defers the 1.5% to `buybackReserve` on any failure.
- If sandwiching becomes material: `setAutoBuybackEnabled(false)` (owner) → the 1.5% accrues to the
  reserve; drain it later with `executeBuybackBurn(minDswpOut, deadline)` using an **off-chain
  computed** `minDswpOut` (never `0` on the bulk path — it is sandwich-exposed).
- Tune `setBuybackSlippageBps(bps)` (≤ `MAX_SLIPPAGE_BPS` = 1000 / 10%) for the inline path.

### 3.5 Funds-stuck / unresolvable dispute
- A dispute with no eligible arbiter freezes (funds stay escrowed — safe). Resolve by ensuring an
  arbiter eligible at that dispute's `disputeRaisedAt` exists, or (for V1) via owner `resolveDispute`.

## 4. Weekly operations dashboard (monitor every week)

| Metric | Source | Healthy |
|---|---|---|
| Active arbiter count | `isArbiter` over the known set / `/v2/admin` | **≥ 3** |
| Pending arbiter queues | `arbiterQueuedAt(addr)` | none unexpected |
| Open disputes & age | subgraph / `getMilestone` status=Disputed | resolved < 7d |
| `buybackReserve` (USDT) | `cast call … "buybackReserve()(uint256)"` | drained periodically |
| Total $DSWP burned | subgraph `GlobalStats.totalDswpBurned` / `/status` | trending up |
| Deployer/keeper BNB balance | `cast balance` | enough for gas |
| `paused` state | `cast call … "paused()(bool)"` | false (unless incident) |
| Owner address | `cast call … "owner()(address)"` | expected multisig/owner |
| DSWP/USDT LP depth (mainnet) | DEX | sufficient before enabling buyback |
| Web app health | `curl -sf https://devswap.pro/en` | 200 |
| CI status | GitHub Actions | green on main |

## 5. Escalation paths

1. **L1 — On-call operator:** triages alerts, runs read-only checks, pauses if a live exploit is
   suspected. Can call `pause()` / `removeArbiter` if holding the owner key (else escalate to L2).
2. **L2 — Owner / multisig signer(s):** ownership actions (pause, transfer, arbiter registry, config).
   Pre-mainnet this is a 3-of-5 multisig + timelock (P5).
3. **L3 — Security lead / auditor:** exploit analysis, postmortem, fix + redeploy decision.
4. **External:** the independent auditor (PeckShield/CertiK) for confirmed vulnerabilities; legal
   counsel for user-fund or regulatory impact.

**Severity → action:** funds-at-risk → L1 pause + page L2/L3 now · degraded (buyback/MEV) → L2 within
24h · cosmetic → normal PR flow.

## 6. Routine operations

- **Buyback burn:** keeper or owner runs `executeBuybackBurn(minDswpOut, deadline)` when
  `buybackReserve > 0` and LP depth is adequate; `minDswpOut` computed off-chain from `getAmountsOut`.
- **Milestone auto-release:** after `reviewTimeout` (7d), the developer or keeper may
  `claimMilestone(jobId, index)` for an unresponsive client.
- **Deploy/verify a new contract:** `forge script script/DeployV2_1.s.sol --rpc-url … --broadcast`
  then `forge verify-contract … --chain 97 --etherscan-api-key …` (key from env, never inline).
  Update `ESCROW_V2` + `NEXT_PUBLIC_ESCROW_V2_*` afterward.
- **Pre-mainnet (P5):** audit clean → owner→multisig+timelock → LP locked (PinkLock) → deploy 56 →
  smoke test. See `SECURITY-AUDIT.md` §5/§6.4.
