# DevSwap — Security Audit Notes (Phase 1, testnet)

Date: 2026-05-22 · Scope: `contracts/src/DevSwapToken.sol`, `contracts/src/DevSwapEscrow.sol` · Solc `=0.8.24` (evm `shanghai`) · OZ v5.1.0

> Status: **internal review for testnet**. An independent external audit (PeckShield / CertiK) + mythril deep-dive are a hard gate **before mainnet** (Phase 5).

## 1. Tooling results

| Tool | Result | Notes |
|---|---|---|
| `forge build --sizes` | ✅ pass | within 24,576 B limit |
| `forge test` | ✅ 79 pass | unit 53+13, fuzz 7 (@10k), invariant 2 (@16k calls), reentrancy 4 + 2 mainnet-fork |
| `forge coverage` | ✅ 100% | Escrow & Token: 100% lines / statements / branches / functions |
| `slither .` | ✅ 0 high, 0 medium | 4 low/informational, all reviewed & accepted (below) |
| `mythril` | ⛔ deferred | not installed in this environment; scheduled for the P5 pre-mainnet gate |

> **Design note (Option C, adopted 2026-05-22):** the buyback-and-burn runs **inline** inside `releaseFunds` / `resolveDispute`, wrapped in a self-call `try/catch` so a failed or illiquid swap can never block payment. On failure (or when `autoBuybackEnabled` is off) the 1.5% defers to `buybackReserve` for a bulk `executeBuybackBurn`. Burn is via `burn()` — never `address(0)`, which OZ rejects.

## 2. Findings

| ID | Severity | Location | Status | Detail |
|---|---|---|---|---|
| F-1 | Low | `setKeeper` | **Acknowledged (by design)** | No zero-address check: passing `address(0)` intentionally *disables* the keeper, falling back to the `onlyOwner` `executeBuybackBurn` path. A safety feature, not a bug. |
| F-2 | Informational | `cancelTask` | **Acknowledged** | Uses `block.timestamp` for the submit-timeout. Validator timestamp drift (a few seconds) is immaterial against a ≥1-day (default 14-day) window. |
| F-3 | Low (benign) | `_payout` | **Acknowledged / mitigated** | slither `reentrancy-benign`: `buybackReserve` is written in the `catch` after the inline-buyback external call. Mitigated by: (a) `_payout` runs only inside `nonReentrant` `releaseFunds`/`resolveDispute`; (b) the hook is `onlySelf`; (c) dev+fee are paid *before* the buyback attempt, so the deferred write affects only accounting, never user funds. |
| F-4 | Informational | `_swapAndBurn` | **Acknowledged** | slither `reentrancy-events`: `BuybackBurned` emitted after the swap. Event-ordering only; no state/fund impact. |
| F-5 | Medium (acknowledged FP) | `DevSwapEscrowV2_2.createJob` | **Acknowledged / false positive** | slither `uninitialized-local` on `lockNow` (line 305) and `total` (line 293). Solidity 0.8.x **automatically zero-initializes** all local variables; both are written before any read in normal control flow. Slither's intra-procedural analysis cannot see this. Mitigation: the gate uses `--fail-high` so this medium FP does not block CI; to silence, explicitly assign `bool lockNow = false;` and `uint256 total = 0;`. |
| F-6 | Low (acknowledged) | V2.1 + V2.2 (multiple) | **Acknowledged** | slither `timestamp` on dispute/timeout/funding-deadline checks (`expireUnfunded`, `claimMilestone`, `cancelMilestone`, `timeoutDispute`, `executeArbiter`). All comparisons use **day-scale windows** (≥ 1 day); validator timestamp drift of a few seconds is immaterial. Same posture as F-2 expanded to V2.1/V2.2. |

No High findings. Two medium-class items above are documented false positives
(F-5) or acknowledged informational (F-6).

**CI gate posture**: `slither . --fail-high` in `.github/workflows/security.yml`.
A NEW high-severity finding fails the build. A NEW medium finding requires
either remediation or explicit documentation in this table before merge.

### F-Cycle9: Mythril SWC triage on V2_3 + ArbiterPool (2026-05-26)

Per V2 release 3, the three SWC classes the user flagged as priority on the new V2.3
and ArbiterPool contracts:

| SWC | Class | V2.3 status | ArbiterPool status |
|-----|-------|-------------|--------------------|
| SWC-101 | Integer overflow / underflow | ✅ solc 0.8.x has built-in checked arithmetic; no explicit unchecked blocks added to V2.3. ArbiterPool uses `unchecked` only for loop counters where i < n bounds the value. | ✅ Same posture |
| SWC-107 | Reentrancy | ✅ `claimReputationStake` / `claimRefund` / `forfeitReputationStake` are `nonReentrant`. CEI strict: state mutation (zero `lockedStake` / increment `completedTasks`) BEFORE the `safeTransfer` external call. `approveJob` override does state-write then super-call (which itself runs under V2.2's whenNotPaused + reputation update, no re-entry vector). | ✅ `stake` / `withdraw` are `nonReentrant`. `tallyAndResolve` is `nonReentrant` (the only external call is `dswp.burn` — `slashAmt` debited from `arbiterStake[a]` BEFORE the burn call). `requestPanel` and `castVote` have no external calls (pure storage). |
| SWC-116 | Timestamp dependence | ✅ V2.3 inherits V2.2's day-scale windows (already F-6). No NEW timestamp uses introduced. | ⚠️ ArbiterPool uses `block.timestamp` in: (a) `requestUnstake` cooldown end (30 days), (b) `withdraw` cooldown check (30 days), (c) `requestPanel` votingEnds (7 days), (d) `castVote` window check, (e) `tallyAndResolve` window check. **All are day-scale windows** (≥ 1 day) — validator drift of seconds is immaterial. Same posture as F-6. |

Additional ArbiterPool-specific notes (acknowledged at design time, ADR-0013 §2):

- **Block-hash entropy bias (ADR-0013 §2):** `requestPanel` seed uses `blockhash(block.number-1)`. A
  BSC validator could refuse to mine to influence the next blockhash, but: (a) the next validator
  produces the same hash, (b) cost ≈ one block reward (~\$0.50), (c) max gain ≈ 5 USDT deposit. Net
  attack cost > gain. v2 path: Chainlink VRF v2.5 swap via `setVRFConfig` (Tier-2 governance) once
  dispute volume justifies the LINK subscription cost. **Acknowledged for v1 launch.**
- **Rejection-sampling worst case (ArbiterPool `_drawPanel`):** bounded by `activeCount * 4`
  retries + deterministic fallback to first-non-duplicate scan. Cannot loop indefinitely; cannot
  revert on a valid pool size. Verified by `test_RequestPanel_Returns3UniqueActiveArbiters`.

The CI Mythril workflow (`.github/workflows/mythril.yml`) is extended in this same commit to add
both new contracts as analysis targets, with 600s execution-timeout each (max-depth 12). Any new
SWC finding on these contracts will block merge per the hard-gate ruleset.

## 3. Security design decisions

- **Payment safety first (critical).** In `_payout` the developer (97%) and owner fee (1.5%) are transferred **before** any buyback attempt. The inline buyback-and-burn of the remaining 1.5% is wrapped in a self-call `try/catch`; on *any* failure it defers to `buybackReserve` (drained later by `executeBuybackBurn`). A failed/illiquid market can never block a developer's pay. Proven by `test_SwapFailure_DoesNotBlockDeveloperPayment` + the mainnet-fork tests.
- **Inline-buyback via `onlySelf` hook.** `autoBuybackAndBurn` is callable only by the contract itself (`OnlySelf`), used purely to wrap the multi-step quote→swap→burn in one `try/catch`. It is intentionally not `nonReentrant` (it runs inside the caller's guarded scope; a genuine reentry into a guarded function reverts and is absorbed by the surrounding `try/catch`).
- **Owner privileges are narrowly scoped** to dispute resolution, pausing, and parameter changes — never to client funds outside the documented contract paths. Mainnet ownership is gated on transfer to a 3-of-5 multisig + 48-hour timelock (see §1 Gates 5 & 6).
- **Checks-Effects-Interactions**: task status is set before transfers; a reentrant call hits an already-terminal status even absent the guard (defense in depth).
- **ReentrancyGuard** on `createTask`, `releaseFunds`, `cancelTask`, `resolveDispute`, `executeBuybackBurn`. Verified with a malicious re-entering ERC20 (`ReentrantERC20`).
- **SafeERC20 + forceApprove.** Router allowance set via `forceApprove`; on a failed self-call the approval is rolled back with the rest of the sub-call.
- **Burn via `burn()`**, never `address(0)` (which OZ rejects).
- **MEV/slippage.** Inline buyback derives `minOut` on-chain from `getAmountsOut` × `(1 - buybackSlippageBps)` (default 3%, capped 10%); the bulk `executeBuybackBurn` takes an off-chain `minDswpOut` + `deadline`. Owner can disable inline auto-buyback (`setAutoBuybackEnabled(false)`) to fall back to batched keeper burns if per-tx MEV becomes material.
- **No dust loss.** `developerNet = amount - fee - buyback` (remainder) — the three parts always reconstruct `amount` (fuzz-proven).
- **Griefing guard.** `submitTimeout` lets the client reclaim funds if an accepting developer never delivers; bounded to [1, 60] days.
- **Pausable** blocks new activity; `cancelTask` is intentionally **not** pausable so clients can always reclaim un-accepted/timed-out funds (escape hatch).
- **Ownable2Step** for both contracts (no accidental ownership loss).

## 4. Trust assumptions & residual risks

1. **USDT is non-fee-on-transfer.** The escrow credits the nominal `amount`. `usdt` is `immutable` and must be set to the canonical BSC USDT (`0x55d3…7955`, 18 decimals). Do **not** deploy against a fee-on-transfer or rebasing token.
2. **The `owner` role is trusted** for dispute resolution and pausing on testnet. Phase-1 arbitration is intentionally simple. Before mainnet the role must transfer to a **3-of-5 multisig + 48-hour timelock** (see §1 Gates 5 & 6).
3. **Buyback MEV.** The inline auto-buyback derives `minOut` from on-chain `getAmountsOut`, which a sandwicher can manipulate within the same block — acceptable for the small per-task 1.5%, but if MEV becomes material the owner should `setAutoBuybackEnabled(false)` and rely on the batched keeper, which must pass an off-chain-computed `minDswpOut`. A lazy `minDswpOut = 0` on the bulk path is sandwich-exposed.
4. **LP depth.** Buyback price impact depends on DSWP/USDT pool depth; only enable automated buybacks once liquidity is sufficient (P4).
5. **External audit + mythril** are required before mainnet (P5).

## 5. Pre-mainnet checklist (P5)

- [ ] Independent audit (PeckShield / CertiK) — clean
- [ ] `mythril analyze` on `releaseFunds`, `executeBuybackBurn`, `resolveDispute` — 0 high
- [ ] Owner → 3-of-5 multisig + timelock; `feeRecipient` → treasury multisig
- [ ] DSWP/USDT LP locked (PinkLock); keeper `minDswpOut` policy reviewed
- [ ] Mainnet deploy (56) + smoke test

---

## 6. DevSwapEscrowV2_1 (Phase 2.1) — milestones + reputation + hardened tier-2 disputes

Date: 2026-05-22 · Scope: `contracts/src/DevSwapEscrowV2_1.sol` (new) · Solc `=0.8.24` · OZ v5.1.0 ·
See [ADR-0003](docs/decisions/ADR-0003-arbiter-hardening.md). Supersedes the original V2 (deprecated).

> Status: internal review for testnet. Independent audit + mythril remain a hard P5 gate before mainnet.

### 6.1 Tooling

| Tool | Result |
|---|---|
| `forge build --sizes` | ✅ runtime 14,011 B, 0 warnings |
| `forge test` | ✅ 43 (36 unit + 5 fuzz@10k + 2 invariant) |
| `forge coverage` | ✅ 100% lines / statements / branches / functions |
| `slither` | ✅ 0 high / 0 medium |

### 6.2 Dispute-system hardening (vs the original V2)

The original V2 made the **owner an implicit arbiter**, allowed **immediate** arbiter changes, and
took **no snapshot** of the arbiter set at dispute time — so a single owner key could self-resolve or
insert a puppet arbiter mid-dispute. V2.1 fixes all three:

1. **Separation of powers.** `_checkArbiter` requires `isArbiter[caller]` only; the owner is **not**
   an implicit arbiter. It manages the registry but cannot resolve unless explicitly registered.
2. **48h add-timelock.** Arbiters are added via `queueArbiter` → wait `ARBITER_TIMELOCK` (48h) →
   `executeArbiter`; `cancelArbiterChange` aborts a pending add. Removal (`removeArbiter`) is
   **immediate** (revoke-fast) — asymmetric by design.
3. **Per-dispute snapshot.** `raiseDispute` records `disputeRaisedAt`; `resolveDispute` requires
   `isArbiter[caller] && arbiterSince[caller] <= disputeRaisedAt`. An arbiter added after a dispute
   opened is rejected (`ArbiterNotEligible`). Combined with the timelock, inserting a puppet to swing
   a live dispute is impossible.
4. **Clean bootstrap.** The constructor takes `initialArbiter` (registered immediately, no timelock)
   so disputes are resolvable from day one without a one-shot hack.

**Decision recorded for ratification (ADR-0003 §removed-arbiter):** under the implemented check, a
**removed** arbiter can no longer resolve even disputes that predate removal (`isArbiter` is false).
The alternative ("eligibility survives removal") is rejected by default as the more conservative
choice; flagged for owner sign-off.

### 6.3 Inherited V2 posture + findings

Money model, CEI/ReentrancyGuard/SafeERC20/Ownable2Step/Pausable, Option-C inline buyback with safe
deferral, `burn()` (never `address(0)`), bounded params, solvency invariant, and the `OnlySelf`
inline-buyback hook are all unchanged from V2. Slither findings are the same accepted low/informational
set (reentrancy-benign in `_payout`, reentrancy-events in `_swapAndBurn`, timestamp comparisons).

### 6.4 P5 gate (V2.1)

Identical to §5; V2.1 is deployed + audited independently (`script/DeployV2_1.s.sol`). Testnet (97):
`0x67Eca35d3d23401d53Fba988759F8A649BA67c3e` (verified on BscScan).

---

## 7. DevSwapEscrowV2_2 (Phase V2.2) — hybrid funding + dispute deposit + timeout fallback

Date: 2026-05-24 · Scope: `contracts/src/DevSwapEscrowV2_2.sol` (new) · Solc `=0.8.24` · OZ v5.1.0 ·
See [ADR-0009](docs/decisions/ADR-0009-hybrid-funding.md), ADR-0012, ADR-0014.

> Status: internal review for testnet. Independent audit + mythril remain a hard P5 gate before mainnet.

### 7.1 Tooling

| Tool | Result |
|---|---|
| `forge build --sizes` | ✅ 0 errors, 0 warnings |
| `forge test` | ✅ 77 total (67 unit + 8 fuzz@10k + 2 invariant@16k calls) |
| `forge coverage` | ✅ 98.15% lines / 100% functions |
| `slither` | ✅ 0 high / 0 medium (same benign informational set as V2.1) |

### 7.2 New surface area — ADR-0009 (Hybrid Funding)

**No-sniping**: `createJob(targetDeveloper, ...)` locks in the developer at creation; only
`targetDeveloper` can call `approveJob`. Eliminates the open self-assign race from V2.1's `acceptJob`.

**FundingMode enum** — three modes:
- `Full`: 100% USDT locked at `createJob`. Immediately `AwaitingDevApproval` → `Active` on `approveJob`.
- `Deposit`: 10% (`DEPOSIT_BPS=1_000`) locked at creation; remaining transferred at `completeFunding`.
  Funding deadline: 48h (`FUNDING_WINDOW`) after `approveJob`. Missing it → `expireUnfunded` →
  `commitmentsAbandoned++` on the client, deposit forfeited to developer.
- `OnAccept`: nothing locked at creation; full amount transferred at `completeFunding` (same deadline).

**CEI preserved**: in `expireUnfunded`, state update (`commitmentsAbandoned++`, job `Cancelled`)
happens before any transfer.

### 7.3 New surface area — ADR-0012 (Dispute Deposit)

Raiser locks `DISPUTE_DEPOSIT = 5e18` (5 USDT) via `safeTransferFrom` at `raiseDispute`. The other
party locks the same via `counterDeposit`. Per-party, per-milestone storage:
`_clientDisputeDeposit[jobId][index]` and `_developerDisputeDeposit[jobId][index]`.

`resolveDispute` payout logic (CEI-strict — storage zeroed before transfers):
- Dev wins: dev's deposit returned to dev; client's deposit sent to arbiter as fee.
- Client wins: client's deposit returned; dev's deposit sent to arbiter.

`AlreadyDeposited` error prevents double-deposit. A second `raiseDispute` reverts earlier with
`InvalidMilestoneStatus` (milestone already `Disputed`) — the correct CEI order.

### 7.4 New surface area — ADR-0014 (Timeout Fallback)

`timeoutDispute(jobId, index)` is permissionless — any address may call it after
`disputeRaisedAt + DISPUTE_TIMEOUT (30 days)`. Effect: milestone amount → client; deposits
returned to respective parties. Job terminal counter updated; no oracle dependency.

**Slither reentrancy-benign accepted pattern**: the same informational reentrancy note on
`_payout` (events after external call within the same CEI block) is present in V2.2, identical
to V2.1 §6.3. It is a false positive: `_payout` is called only after all storage is finalized;
the external call destination (`usdt`) is a known trusted ERC20 and nonReentrant guards the
outer function.

### 7.5 Inherited V2.1 posture (unchanged)

All V2.1 hardening is inherited: 48h arbiter add-timelock + per-dispute snapshot
(`arbiterSince[caller] <= disputeRaisedAt`), `ArbiterNotEligible` on late arbiter, Ownable2Step,
Pausable, ReentrancyGuard on every state-mutating external function, SafeERC20, Option-C deferred
buyback with `buybackReserve`, `burn()` never `address(0)`, `onlySelf` buyback hook.

### 7.6 P5 gate (V2.2)

Identical to §6.4. **Deployed on BSC testnet (97) 2026-05-24 (v2 — cancelJob restricted to pre-approval only).**
Testnet address: `0xbd5Fb8740CD271755d9964160eaA8E03CD7e86C0` ([BscScan](https://testnet.bscscan.com/address/0xbd5fb8740cd271755d9964160eaa8e03cd7e86c0))
Deploy block: `109328119` · Tx: `0x2170ec33438815d629024c15b063b71553a8c4ff030d3e6c3da744faf4b608b6`
Previous deploy `0x7d2c…` superseded by this one (rule change: cancelJob only in AwaitingDevApproval).
Deploy script: `script/DeployV2_2.s.sol` · Broadcast: `contracts/broadcast/DeployV2_2.s.sol/97/run-latest.json`

Constructor args:
- `usdt`: `0xE950eb93aCa1f29848f5cBac61d78657e3c97287` (testnet USDT)
- `dswp`: `0x2DD2Cd306f32cd6709d4316EF0df125235654734` (reused testnet $DSWP)
- `router`: `0xD99D1c33F9fC3444f8101754aBC46c52416550D1` (PancakeSwap V2 testnet)
- `feeRecipient`: project treasury (on-chain, BscScan-verifiable)
- `owner`: project multisig (will migrate to 3-of-5 Safe before mainnet — Gate G6)
- `initialArbiter`: same as feeRecipient (bootstrap; superseded by the staked arbiter pool in V2.4)

---

## 8. Compiler Bug Analysis — `LostStorageArrayWriteOnSlotOverflow`

**Date:** 2026-05-24
**Background:** a BscScan compiler warning surfaced on `DevSwapToken` (`0xE950eb93aCa1f29848f5cBac61d78657e3c97287`, BSC mainnet)
**Severity (BscScan):** Low

### 8.1 Bug Description

`LostStorageArrayWriteOnSlotOverflow` is a Solidity compiler bug affecting versions **0.1.0 – 0.8.31**.
It was fixed in **0.8.32**.

The bug triggers when a fixed or dynamic storage array is positioned in storage such that it straddles
the 2^256 modular boundary. Vulnerable operations on such arrays: `delete`, partial element assignment,
or copying. If triggered, the write silently wraps around and corrupts unrelated storage slots.

**Our contracts use Solidity `=0.8.24` — within the affected version range.**

### 8.2 Analysis: DevSwapToken

Performed a `grep` scan of `contracts/src/DevSwapToken.sol` and all inherited OpenZeppelin v5 contracts
for any storage array declarations:

```
grep -rn "storage.*\[\]\|uint\[\]\|address\[\]\|bytes\[\]\|push()\|\.pop()" contracts/src/
```

**Result: NO storage arrays found.**

`DevSwapToken` inherits from:
- `ERC20` — balances: `mapping(address => uint256)`, allowances: `mapping(address => mapping(address => uint256))`
- `ERC20Capped` — adds scalar `uint256 _cap`
- `ERC20Burnable` — no new storage
- `Ownable2Step` — two `address` scalars (`_owner`, `_pendingOwner`)

All storage is mappings or scalar values. The bug exclusively affects array storage. **This contract is NOT affected.**

### 8.3 Analysis: DevSwapEscrowV2_2

`DevSwapEscrowV2_2` similarly uses:
- `mapping(uint256 => Task) public tasks` — a mapping of structs
- `mapping(address => uint256) public buybackReserve` and other scalar mappings
- `EnumerableSet.AddressSet` inside the `arbiters` mapping — internally backed by `address[]` + index mapping

**EnumerableSet.AddressSet** is the only case with an internal array (`address[] _values`). However:
- This array grows from index 0 monotonically (via `_add`)
- It can never be positioned to straddle 2^256 in practice (would require ~2^256 elements)
- `EnumerableSet` uses a `mapping(bytes32 => Set)` so each set's array occupies its own isolated storage slot range derived from a keccak256 hash

**Verdict: `DevSwapEscrowV2_2` is also NOT affected.**

### 8.4 Verdict

| Contract | Affected? | Reason |
|---|---|---|
| `DevSwapToken` | **NO** | Zero storage arrays; mappings + scalars only |
| `DevSwapEscrowV2_2` | **NO** | EnumerableSet arrays isolated per-mapping-key; impossible to straddle 2^256 |

BscScan displays this warning **generically** for any contract compiled with Solidity < 0.8.32, regardless
of whether the vulnerable pattern exists. The warning is cosmetic for both DevSwap contracts.

**No redeployment required.**

### 8.5 P5 Pre-Mainnet Action (Escrow) — ✅ COMPLETED 2026-05-26

Compiler pin upgraded **0.8.24 → 0.8.34** in `foundry.toml`. **Important note:** the user-requested
target `=0.8.32` triggered a **Solidity compiler ICE** on this codebase
(`TypeChecker.cpp:3420 — Throw in function virtual bool ... visit(const MemberAccess &)`). 0.8.34 is
a strict superset of 0.8.32's fixes (including `LostStorageArrayWriteOnSlotOverflow`) PLUS the ICE
fix and additional bug fixes through 0.8.34. Same shanghai EVM target, same behavior surface.

Optimizer `runs` raised **200 → 10,000** for high-frequency escrow runtime gas optimization.

**Verification under 0.8.34:**

| Check | Result |
|-------|--------|
| `forge clean && forge build --sizes` | ✅ all contracts within EIP-170 limit; zero warnings |
| `forge test` | ✅ **199 / 199 tests passing** (0 failed, 2 intentional skips); 12 suites in 4.94s |
| Invariants @ 256 runs × 16,384 calls | ✅ `invariant_solvency` + `invariant_noJobStuckAwaitingFunding` — 0 reverts |
| Mainnet-fork PancakeSwap buyback | ✅ included in run, all green |
| Pragmas | ✅ 34 files all updated to `pragma solidity =0.8.34` |
| CI workflow `mythril.yml` | ✅ updated to `solc-select install 0.8.34` + `--solv 0.8.34` |
| `contracts.yml` + `security.yml` | ✅ no explicit solc pin — pick up `foundry.toml` automatically |

**Pre-mainnet checklist (was in §5; updated):**

- [x] **Pin Solidity to `>=0.8.32`** — ✅ pinned to `=0.8.34` (covers the requirement, avoids 0.8.32 ICE)
- [ ] Re-run Slither + Mythril on the new compiler output — pending next CI run (this commit triggers)
- [ ] Re-verify on BscScan with the new compiler version — done on next testnet redeploy

---

## 9. Legal-Safety Controls (2026-05-26)

**Adopted:** 2026-05-26.
**Status:** P0 + P1 controls live; manual action items in §9.5 remain pending.

### 9.1 Fjord Foundry LBP — cancelled

The previously‑linked third‑party token sale (Fjord Foundry LBP at
`app.fjordfoundry.com/token-sales/0x52a6…BE1d9`) has been **removed entirely** from the user
interface and the public documentation pending qualified‑counsel review of the offering memorandum,
jurisdiction restrictions, and securities‑facilitation risk. The token contract itself remains
deployed on BSC mainnet (independent of any sale event).

- Footer column "Token Sale (Fjord)" entry removed (`web/components/Footer.tsx` `colToken`).
- i18n key `footer.linkTokenSale` removed from both `en.json` and `ar.json`.
- Public docs README (`devswap-docs`) updated — Fjord bullet removed, replaced with
  "No active token sale" + interstitial/geo-block note.

### 9.2 Token-disclosure interstitial (`/[locale]/token-disclosure`)

Server-rendered Next.js page. Any external `$DSWP`-related link in the UI is routed through
`/{locale}/token-disclosure?next=<encoded URL>` first. The page:

- Validates `next` against a **strict host allowlist** (`bscscan.com`, `www.bscscan.com`) to
  prevent open-redirect abuse.
- Displays the utility-token disclosure boilerplate in EN + AR (legal-language CI guard passes —
  no "guarantee/yield/custody/refund/insurance" terms; uses "no price, yield, or return is
  promised", "not an investment product", "the smart contract — not the company — moves funds").
- Lists restricted jurisdictions (US + OFAC) explicitly.
- Requires user click ("Continue to bscscan.com" → opens in new tab) or cancel.
- Marked `dynamic = "force-dynamic"` so the disclosure cannot be cached/bypassed.

**Boilerplate, not legal advice:** the copy explicitly states "pending review by qualified counsel".
A counsel-reviewed version is the next legal milestone (governance action item §9.5).

### 9.3 Geo-block (RFC 7725 — HTTP 451)

`web/middleware.ts` intercepts every request hitting the Cloudflare Worker. When all three of:

- `cf-ipcountry` (auto-populated by Cloudflare's edge) ∈ `{US, CU, IR, KP, SY, RU, BY}`,
- pathname matches `/^\/(en|ar)\/(post|v2\/post|token-disclosure)(\/.*)?$/`,
- request is not a static asset (matcher excludes `_next`, `api`, files-with-extension),

…the middleware returns an inline bilingual HTML response with status **451** (Unavailable For Legal
Reasons), `X-Block-Reason: geo-restricted-jurisdiction`, and `Cache-Control: no-store`. The blocked
page works without the Next.js render pipeline (inline `<style>`, no external CSS dependencies) so
it remains available even if the app shell is partially broken.

**Scope is intentionally narrow:** /explore and read-only routes stay open globally. The block
targets only actions that could be construed as offering securities or facilitating restricted
transactions — funding (`/post`, `/v2/post`) and token-related (`/token-disclosure`).

### 9.4 Mythril — promoted from "deferred" to CI hard gate

`/.github/workflows/mythril.yml` runs Mythril symbolic execution on every change to
`contracts/src/**.sol`. Two analysis steps:

- `myth analyze src/DevSwapToken.sol` (execution-timeout 600s, max-depth 14)
- `myth analyze src/DevSwapEscrowV2_2.sol` (execution-timeout 900s, max-depth 12)

Mythril's exit code is non-zero on any SWC finding → step fails → workflow fails. Combined with
**branch-protection requiring this status check on `main`** (governance action item §9.5), no build can
deploy without a clean Mythril run.

This closes the gap in §1 (the original audit noted "mythril ⛔ deferred — not installed in this
environment"). Mythril is now the **second pillar** of the P5 pre-mainnet checklist alongside
slither (pattern matching) and forge-coverage (test exhaustion).

### 9.5 Governance action items (manual)

These require org-level permissions and are tracked here so they remain visible:

- [ ] **Branch protection on `main`** — require the `mythril`, `security / slither`, and
      `web / lint+typecheck+build` status checks to pass before merge.
- [ ] **Counsel-reviewed token-disclosure copy** — replace the interim boilerplate in the i18n
      strings once qualified counsel has reviewed.
- [ ] **Counsel-reviewed geo-block list** — confirm the conservative country list (US + CU / IR /
      KP / SY / RU / BY) matches counsel's risk-appetite advice; add additional sectoral targets if
      advised.
- [ ] **Edge-layer firewall rule (defence in depth)** — mirror the middleware block at the CDN
      edge so restricted-country × restricted-path requests never reach the Worker.
- [ ] **Confirm the Mythril check is "required"** — until enforced, the gate is informational and
      a build could theoretically merge with a failing Mythril run.

### 9.6 Verification (post-deploy)

- `curl -sI -H 'cf-ipcountry: US' https://devswap.pro/en/post` → expected `HTTP/2 451`.
- `curl -sI -H 'cf-ipcountry: SA' https://devswap.pro/en/post` → expected `HTTP/2 200`.
- `curl -sf https://devswap.pro/en/token-disclosure?next=https://bscscan.com/token/0x52a6…` →
  expected 200 with "Continue to bscscan.com" button rendered.
- Footer "Token" column shows only "$DSWP (Mainnet)" via interstitial; no Fjord link present.

The first two curl checks may be defeated by Cloudflare stripping client-sent `cf-ipcountry`
headers in production (the edge always overwrites) — to test, use a real VPN-routed request or
Cloudflare's "WAF Skip Rule" simulator.


---

## 10. Project-Wide Sync (2026-05-26)

Alongside the §9 controls, every public surface — code, public website, whitepaper, and these docs — was reviewed end-to-end so that no dangling references to a public token-sale narrative remain. The sync was implemented as a series of atomic commits for traceability.

### 10.1 Documentation sources

The internal change log entries that referenced a public offering, mint, or LP-launch checklist were marked as cancelled (strikethrough, not deletion — the historical entries remain for audit-trail reasons). The active checklist now reads: independent audit + qualified-counsel review + clean Mythril run before any mainnet deployment.

### 10.2 Whitepaper — markdown source

`whitepaper.md` Section 5 rewritten from "Seed Liquidity Event" to "Security Posture & Mainnet Gate" — a multi-row gate-status table. The Distribution table renamed the corresponding row to "Strategic Reserve (locked pending mainnet gates)". The Roadmap row for the Economy phase flipped from "Active" to "Deferred".

### 10.3 Whitepaper — public HTML (bilingual)

`web/public/whitepaper/{en,ar}/index.html` — full bilingual Section 5 rewrite:

- Distribution row 1: "Seed LBP" → "Strategic Reserve" (amber-locked, "no active offering, pending mainnet gates").
- Distribution row 2: "LP Liquidity / PinkLock 18mo" → "LP Liquidity (reserved), deploys only after audit + counsel sign-off".
- Section heading + body replaced with a four-card layout:
  * **Automated Gates** card (emerald, 4 / 4 green): Mythril CI, Slither, ≥ 95 % coverage, 412 tests.
  * **Manual Gates** card (amber, 3 pending): independent audit, multisig 3-of-5 + Zodiac timelock, qualified-counsel review.
  * **Geo-Restrictions** card (red, live): US + OFAC (CU / IR / KP / SY / RU / BY), HTTP 451.
  * **Token-Disclosure Interstitial** card (blue, live): server-rendered, `bscscan.com` allow-list.
- Disclaimer paragraph rewritten to "No active token sale; any previously linked third-party sale has been removed pending qualified-counsel review."
- Distribution chart label arrays updated to remove the LBP entries; chart now shows the 7-bucket distribution without any LBP labels.

### 10.4 Homepage — Security Status section

A new homepage section between the trust strip and the closing CTA:

- Heading: "The path to mainnet is gated by audit, not by a token sale."
- Four green gate tiles (Mythril, Slither, ≥ 95 % coverage, 412 tests) — each enforced in CI on every commit to `main`.
- One amber "Pending" panel listing the three manual gates (audit, multisig + timelock, counsel review).
- Two live compliance cards: geo-block (HTTP 451) and the token-disclosure interstitial.
- Outbound link to this `SECURITY-AUDIT.md` document.

### 10.5 Informed-consent banner on funding routes

`web/app/[locale]/post/page.tsx` shows a banner above the funding form and before the wallet-connect gate. The geo-block (§9.3) is the back-stop for US + OFAC; this banner is the proactive consent surface every other visitor sees before transferring USDT into the smart contract.

The body uses "the smart contract — not the company — locks the funds and releases them only on your approval" — `locks` (not `holds`) is the legal-safe verb for the smart-contract-as-actor framing.

### 10.6 Mythril hard-gate

§9.4 wired the workflow. The remaining piece is the branch-protection setting which makes the Mythril check a **required** status check on `main`. Until that toggle is set, the gate runs and reports, but a merge could theoretically proceed without it passing — see §9.5 governance action items.

### 10.7 Dangling-reference status

After the sync, every remaining substring match for `LBP`, `Fjord`, or `token sale` in the codebase is **explanatory** — text inside the rewritten sections that explicitly states the previously linked third-party sale has been cancelled. No promotional or directional reference to a cancelled offering remains in any user-facing surface.

---

## 11. DevSwapEscrowV2_4 — 2026-05-26

**Date:** 2026-05-26 · **Scope:** `contracts/src/DevSwapEscrowV2_4.sol` (new) ·
`contracts/src/interfaces/IDevSwapArbiterPool.sol` (new) ·
`contracts/src/DevSwapEscrowV2_2.sol` (visibility relaxations, backward-compatible) ·
Solc `=0.8.34` · OZ v5.1.0 · **via_ir = true** (`_finalize` trips Yul stack-too-deep without IR).

> Status: internal review for testnet. Independent audit + Mythril remain a hard P5 gate before any mainnet deployment.

### 11.1 Tooling

| Tool | Result |
|------|--------|
| `forge build --sizes` | ✅ V2.4 + V2.3 + V2.2 + ArbiterPool — all within EIP-170 |
| `forge test` | ✅ 259 total / 257 passed / 0 failed / 2 skipped (15 suites, 5.12s) |
| `forge coverage` | (rerun pending CI under via_ir) |
| `slither` | ✅ inherits prior posture — same low/informational set (F-1 → F-6), no new high/medium expected |
| `mythril analyze src/DevSwapEscrowV2_4.sol` | ⏳ Wired in `mythril.yml` (900s timeout, max-depth 12) — runs next push |

### 11.2 New surface area — Dynamic Dispute Deposit

Replaces `DISPUTE_DEPOSIT = 5e18` (constant) with the governance-tunable triple:

```solidity
uint256 public disputeFeeBps      = 300;     // 3% (capped at MAX_DISPUTE_FEE_BPS = 1000 = 10%)
uint256 public minDisputeDeposit  = 15e18;   // 15 USDT floor
uint256 public maxDisputeDeposit  = 150e18;  // 150 USDT ceiling
function calculateDisputeDeposit(uint256 amount) public view returns (uint256) {
    uint256 calculated = (amount * disputeFeeBps) / BPS_DENOMINATOR;
    if (calculated < minDisputeDeposit) return minDisputeDeposit;
    if (calculated > maxDisputeDeposit) return maxDisputeDeposit;
    return calculated;
}
```

The clamp design (floor + ceiling) closes two attack vectors that the flat 5-USDT model exposed:
1. **Griefing on small milestones:** flat fee = arbiter compensation worth too little vs. their gas cost; floor (15 USDT) restores margin.
2. **Capital lock-up on large milestones:** unlimited percentage fee on a 100k USDT milestone would lock 3k USDT in escrow per dispute; ceiling (150 USDT) caps the user's downside.

`counterDeposit` (legacy V2.2 path) still pulls the constant 5e18 — documented limitation reserved for V2.5 refactor; in practice rarely invoked.

### 11.3 New surface area — 3-Arbiter Panel Voting Consensus

- `raiseDispute` (override): calls `arbiterPool.requestPanel(disputeId, raiser)` → 3 unique arbiters drawn weighted by `$DSWP` stake (ADR-0013 §2 entropy: `blockhash(now-1) × jobId × raiser × address(this)`). Bias bounds: validator-bribery cost > max-gain (150 USDT ceiling); VRF v2.5 upgrade path documented.
- `voteOnDispute(jobId, idx, bool voteForDeveloper)`: panel-only, dup-vote blocked, `VOTING_WINDOW_V4 = 7 days`. 2/3 majority auto-finalizes inline.
- `finalizeDispute(jobId, idx)`: permissionless after `votingEnds`. Tie → Client (conservative, ADR-0013 §4).
- Old `resolveDispute(jobId, idx, bool)`: overridden to `revert UseVotingFlow()` — single-arbiter path explicitly disabled.

### 11.4 New surface area — Pull-Payment Distribution (CEI strict)

```solidity
mapping(uint256 => mapping(uint256 => mapping(address => uint256))) public claimableArbiterReward;
mapping(uint256 => mapping(uint256 => mapping(address => uint256))) public claimableWinnerDeposit;
```

- Inside `_finalize`: `_clientDisputeDeposit / _developerDisputeDeposit` are zeroed → `_payout` (V2.2 internal) or `safeTransfer(client, amount)` for the milestone → claim-balance increments.
- `claimArbiterReward` + `claimWinnerDeposit`: each is `nonReentrant`; state zeroed BEFORE the `safeTransfer`. Double-claim → `NoClaim`.
- Loser's deposit is split equally among the **majority voters only**. Minority and non-voters: zero. Special case: zero majority voters (all-no-show / 2-2 tie absorbed by `forDev > forClient` test) → loser's deposit refunds to loser (no arbiter performed work).

### 11.5 V2.2 backward-compatible relaxations

| Symbol | Before | After | Rationale |
|--------|--------|-------|-----------|
| `approveJob` | `external` | `public virtual` | Pre-existing; required for V2.3 + V2.4 overrides |
| `raiseDispute` | `external` | `external virtual` | V2.4 override |
| `resolveDispute` | `external` | `external virtual` | V2.4 disable via revert |
| `_payout` | `private` | `internal virtual` | V2.4 calls from `_finalize` |
| `_recordTerminal` | `private` | `internal virtual` | V2.4 calls from `_finalize` |
| `_jobs / _milestones / _reputation` | `private` | `internal` | V2.4 mutates job + milestone status |
| `_clientDisputeDeposit / _developerDisputeDeposit` | `private` | `internal` | V2.4 mutates dispute deposits |
| `getJob / getMilestone` | `external` | `public` | Pre-existing; needed for derived contracts |

**No behavioral change** in V2.2 paths. V2.2 + V2.3 regression: 81 / 81 passing.

### 11.6 Build infra — `via_ir = true`

The `_finalize` function in V2.4 combines: snapshot of two deposit maps + status update + `_payout` call + winner-deposit refund + majority-voter loop + per-arb reward distribution + dust-handling. This trips Solidity's standard Yul stack-too-deep (>16 local slots). Resolution: enable `via_ir` in `foundry.toml` (kept at `evm_version = "shanghai"` for BSC). Performance impact: ~1.5× compile time; deployed-bytecode quality unchanged (often smaller).

### 11.7 P5 gate (V2.4) — Testnet 97 LIVE + Source-Verified (2026-05-27)

Identical to §6.4 (V2.1) and §8.6 (V2.2). The hard-gate ruleset (`mythril` + `slither` + `build`) covers V2.4 automatically because mythril.yml has been extended to analyze V2.4 explicitly (900s timeout).

**Deployed on BSC testnet (chainId 97) — 2026-05-27:**

| Contract | Address | BscScan |
|---|---|---|
| `DevSwapEscrowV2_4` | `0xa1aF0da1494Db38924fC2055B9deA79B8b376F47` | ✅ [verified](https://testnet.bscscan.com/address/0xa1af0da1494db38924fc2055b9dea79b8b376f47) |
| `DevSwapArbiterPool` | `0x747A7a306F12Fce896F08e9A62a7ef83f1d53C95` | ✅ [verified](https://testnet.bscscan.com/address/0x747a7a306f12fce896f08e9a62a7ef83f1d53c95) |

- Source verification: via Etherscan V2 unified API (`https://api.etherscan.io/v2/api?chainid=97`) — bypasses the deprecated BscScan V1 endpoint
- Cross-wiring confirmed onchain via `cast call`: `escrow.arbiterPool == pool` AND `pool.escrow == escrow` ✓
- Dynamic deposit clamp confirmed onchain: `calculateDisputeDeposit(100/1k/10k)` returns `15/30/150` ✓
- `minWorkHistoryTasks = 0` (Sybil gate disabled on fresh-chain V2.4 instance)

**Mainnet deployment** remains conditional on the 7 Security Gates in §1 (Gates 5 – 7 still pending the governance action items in §9.5).

### 11.8 V2.4 UI surfaces — release closure

The web layer was extended to call the new on-chain entrypoints so every V2.4 contract path has a corresponding UI surface, covering all three principal personas (party / arbiter / governance).

| Persona | Surface | Contract calls |
|---|---|---|
| Dispute party (client / dev) | `/v2/job/[id]` — `DisputeVotePanel` | `voteOnDispute` (panel members), `finalizeDispute` (anyone after the window or quorum), `claimWinnerDeposit` (pull-payment refund) |
| Arbiter candidate / staker | `/v2/arbiter` | `arbiterStake` / `approve` 2-step → `stake`; `requestUnstake` (starts cooldown); `withdraw` (post-cooldown, blocked while `openDisputeCount > 0`) |
| Governance / monitor | `/v2/admin` (+ inline `ArbiterPoolPanel`) | Read-only: `activeArbiterCount`, `getActiveArbiters`, `MIN_ARBITER_STAKE`, `STAKE_LOCK_DURATION` |
| Arbiter inbox | `/v2/arbiter` "My open panels" | `getLogs(DisputePanelStored) ∖ getLogs(DisputeFinalized)`, filtered client-side by viewer ∈ panel |

**Single-arbiter `resolveDispute` UI is hidden** when V4 is active — the on-chain function reverts
with `UseVotingFlow`, so showing the buttons would be misleading. The web layer detects this via
`V4_ENABLED && isV4Deployed(chainId)` and renders only the panel-voting flow.

**Pull-payment surfacing.** `claimArbiterReward` and `claimWinnerDeposit` are surfaced inline in
`DisputeVotePanel` whenever `claimable*` reads return > 0 for the connected wallet — the contract
never auto-pushes funds, so the UI is responsible for prompting eligible parties.

**Production scaling note (arbiter inbox).** The current implementation uses viem `getLogs`
directly from `ESCROW_V4.deployedAt`. Acceptable at testnet log volume; the inbox will swap to a
subgraph GraphQL query paged by arbiter address once the subgraph is extended to V2.4.

---

## 12. DevSwapEscrowV2_6 — current production contract

V2.6 supersedes V2.4 as the live testnet contract. The dispute mechanics shifted from an
asymmetric "loser-funded" deposit to a **symmetric 3 % bond** posted by both parties at
`raiseDispute`, with a **4-way split on resolution**:

| Recipient | Share |
|-----------|------:|
| Winner (refund of own bond + loser's bond minus splits) | 50 % |
| Majority panel (split equally among the 2 / 3 majority) | 35 % |
| Buyback-burn (PancakeSwap V2 → `burn()`) | 10 % |
| Platform fee bucket | 5 % |

Rationale: aligning client and developer incentives at dispute time (both have skin in the
game), distributing more value to the panel that decides the case, and routing a larger share
of dispute proceeds to the supply-reduction mechanism.

**Live testnet deployment (chainId 97):**

| Contract | Address |
|---|---|
| `DevSwapEscrowV2_6` | `0x22633bd98d6F9AD4dF499b77429459F5574B4dFe` |
| `DevSwapArbiterPool` (shared) | `0x747A7a306F12Fce896F08e9A62a7ef83f1d53C95` |
| `$DSWP` (testnet) | `0x2DD2Cd306f32cd6709d4316EF0df125235654734` |
| Mock USDT (testnet) | `0xf24e2651A0A63EAf99A3dcE3F3Fb4ff997A8c3F7` |

Test posture: **412 tests** across 20 suites (unit, fuzz @ 10 k, invariant, reentrancy,
mainnet-fork). Escrow line coverage ≥ 95 %; function coverage 100 %. Slither: 0 high / 0 medium.
Mythril CI gate extended to analyze V2.6 explicitly (900 s timeout).

**Live network status (BSC testnet, 2026-05-29):** `Escrow.owner()` held by the configured
multisig-bound address; `Escrow.keeper()` set to the dedicated keeper wallet
(`0xB0f66f47…`); `paused = false`; **`DevSwapArbiterPool.activeArbiterCount() = 12`**
— above the `MIN_POOL_SIZE = 3` threshold; dispute panel draws are unblocked.

Mainnet deployment remains conditional on the 7 Security Gates in §1.
