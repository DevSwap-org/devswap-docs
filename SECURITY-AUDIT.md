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

> **Design note (Option C, owner-approved 2026-05-22):** the buyback-and-burn now runs **inline** inside `releaseFunds`/`resolveDispute` (per owner directive), wrapped in a self-call `try/catch` so a failed/illiquid swap can never block payment; on failure (or when `autoBuybackEnabled` is off) the 1.5% defers to `buybackReserve` for a bulk `executeBuybackBurn`. Burn is via `burn()` (never `address(0)`, which OZ rejects).

## 2. Findings

| ID | Severity | Location | Status | Detail |
|---|---|---|---|---|
| F-1 | Low | `setKeeper` | **Acknowledged (by design)** | No zero-address check: passing `address(0)` intentionally *disables* the keeper, falling back to owner-only `executeBuybackBurn`. A safety feature, not a bug. |
| F-2 | Informational | `cancelTask` | **Acknowledged** | Uses `block.timestamp` for the submit-timeout. Validator timestamp drift (a few seconds) is immaterial against a ≥1-day (default 14-day) window. |
| F-3 | Low (benign) | `_payout` | **Acknowledged / mitigated** | slither `reentrancy-benign`: `buybackReserve` is written in the `catch` after the inline-buyback external call. Mitigated by: (a) `_payout` runs only inside `nonReentrant` `releaseFunds`/`resolveDispute`; (b) the hook is `onlySelf`; (c) dev+fee are paid *before* the buyback attempt, so the deferred write affects only accounting, never user funds. |
| F-4 | Informational | `_swapAndBurn` | **Acknowledged** | slither `reentrancy-events`: `BuybackBurned` emitted after the swap. Event-ordering only; no state/fund impact. |

No High or Medium findings.

## 3. Security design decisions

- **Payment safety first (critical).** In `_payout` the developer (97%) and owner fee (1.5%) are transferred **before** any buyback attempt. The inline buyback-and-burn of the remaining 1.5% is wrapped in a self-call `try/catch`; on *any* failure it defers to `buybackReserve` (drained later by `executeBuybackBurn`). A failed/illiquid market can never block a developer's pay. Proven by `test_SwapFailure_DoesNotBlockDeveloperPayment` + the mainnet-fork tests.
- **Inline-buyback via `onlySelf` hook.** `autoBuybackAndBurn` is callable only by the contract itself (`OnlySelf`), used purely to wrap the multi-step quote→swap→burn in one `try/catch`. It is intentionally not `nonReentrant` (it runs inside the caller's guarded scope; a genuine reentry into a guarded function reverts and is absorbed by the surrounding `try/catch`).
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
2. **Owner is trusted** for dispute resolution and pausing. Phase-1 arbitration is intentionally simple. Before mainnet the owner must become a **3-of-5 multisig + timelock** (P5).
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
- `feeRecipient`: `0xb60Eef021F26F5186b54c1c3E872ea5db5c32e61`
- `owner`: `0x1965E194C1A8f2574c41B2894F5cDf1bD2726e59`
- `initialArbiter`: `0xb60Eef021F26F5186b54c1c3E872ea5db5c32e61` (= feeRecipient)

---

## 8. Compiler Bug Analysis — `LostStorageArrayWriteOnSlotOverflow`

**Date:** 2026-05-24
**Trigger:** BscScan compiler warning on `DevSwapToken` (`0xE950eb93aCa1f29848f5cBac61d78657e3c97287`, BSC mainnet)
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

**Conclusion: DevSwapEscrowV2_2 is also NOT affected.**

### 8.4 Verdict

| Contract | Affected? | Reason |
|---|---|---|
| `DevSwapToken` | **NO** | Zero storage arrays; mappings + scalars only |
| `DevSwapEscrowV2_2` | **NO** | EnumerableSet arrays isolated per-mapping-key; impossible to straddle 2^256 |

BscScan displays this warning **generically** for any contract compiled with Solidity < 0.8.32, regardless
of whether the vulnerable pattern exists. The warning is cosmetic for both DevSwap contracts.

**No redeployment required.**

### 8.5 P5 Pre-Mainnet Action (Escrow)

When deploying `DevSwapEscrowV2_2` to BSC mainnet (P5 gate), upgrade the compiler pin to `=0.8.32` (or
latest stable at that time) to eliminate this warning cleanly and pick up all bug fixes between 0.8.24
and 0.8.32. Run the full test suite after the bump — no API surface changes are expected.

Add to the P5 pre-mainnet checklist in `BLOCKED.md`:
- [ ] Pin Solidity to `>=0.8.32` for all mainnet-deployed contracts
- [ ] Re-run slither + mythril on the new compiler output
- [ ] Re-verify on BscScan with the new compiler version

---

## 9. Legal-Safety Controls (2026-05-26)

**Date:** 2026-05-26 · **Decision:** owner directive after critical-review of generic AI advice that
missed the actual gaps in DevSwap's posture (geo-block, token-sale disclosure, mythril deferral).
**Status:** P0 + P1 implemented in commit `<this commit>`; owner action items below remain.

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
A counsel-reviewed version is the next legal milestone (owner action item §9.5).

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
**branch-protection requiring this status check on `main`** (owner action item §9.5), no build can
deploy without a clean Mythril run.

This closes the gap in §1 (the original audit noted "mythril ⛔ deferred — not installed in this
environment"). Mythril is now the **second pillar** of the P5 pre-mainnet checklist alongside
slither (pattern matching) and forge-coverage (test exhaustion).

### 9.5 Owner action items (manual; not Claude-executable)

These require GitHub-org-level permissions and are tracked here so they don't get lost:

- [ ] **Branch protection on `main`** — Settings → Branches → `main` → Require status checks to
      pass before merging → add `mythril (P5 hard gate) / mythril` AND `security / slither` AND
      `web / lint+typecheck+build` to required.
- [ ] **Counsel-reviewed token-disclosure copy** — replace the boilerplate in
      `web/messages/{en,ar}.json` `tokenDisclosure.*` once qualified counsel has reviewed.
- [ ] **Counsel-reviewed geo-block list** — confirm the conservative country list (US + CU/IR/KP/SY +
      RU/BY) matches the firm's risk appetite; add VE or other sectoral targets if advised.
- [ ] **Cloudflare WAF rule (optional, defence in depth)** — add a CF Firewall Rule at the
      edge that mirrors the middleware block, so the request never reaches the Worker for
      restricted countries × paths.
- [ ] **Verify Mythril CI is "required" on org settings** — until set, the gate is informational
      and a build *could* be merged with failing Mythril.

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

## 10. System-Wide Sync (2026-05-26)

**Trigger:** owner directive — after the §9 P0+P1 implementation, conduct a full sync of every
project surface (code, journals, public website, whitepaper) so no dangling references to the
cancelled Fjord LBP / token-sale narrative remain. Implemented as a series of atomic
task-commit-deploy cycles for traceability.

### 10.1 Journals (commit `ec56f6e`)

`TASKS.md`, `STATE.md`, `BLOCKED.md` — Fjord LBP / Mint 4M / PancakeSwap LP tasks marked
`[X CANCELLED 2026-05-26]` (strikethrough, not deletion — preserves audit trail). New top-of-file
entry in `STATE.md` documents the pivot to "Security-Hardened P0" and references the four §9
controls. `BLOCKED.md` §4 reframed: "audit + counsel + Mythril clean" gating replaces the
LBP-launch checklist.

### 10.2 Whitepaper — markdown sources (commit `33c9d02`)

`docs/whitepaper.md` Section 5 rewritten from "Seed Liquidity Event (Fjord LBP)" to
"Security Posture & Mainnet Gate (P5)" — a 9-row gate-status table. Tokenomics row 1 relabeled
from "Seed LBP" to "Strategic Reserve (locked pending P5 gates)". Roadmap P4 status flipped from
"Active" to "Deferred 2026-05-26". (`whitepaper-ar-en.md` at repo root is gitignored; edited
locally for the owner's working copy only.)

### 10.3 Whitepaper — public HTML (commit `bbe2a09`, deploy `v 106994f9`)

`web/public/whitepaper/{en,ar}/index.html` — full bilingual Section 5 rewrite:

- Distribution table row 1: "Seed LBP" → "Strategic Reserve" (amber-locked, "Held by deployer,
  no active offering, pending P5 gates").
- Distribution row 2: "LP Liquidity / PinkLock 18mo" → "LP Liquidity (reserved), deploys only
  after P5 audit + counsel sign-off".
- Section heading + body replaced with a four-card layout:
  * **Automated Gates** card (emerald, 4/4 green): Mythril CI, Slither, 100% coverage, 79+ tests.
  * **Manual Gates** card (amber, 3 pending): independent audit, multisig 3-of-5 + Zodiac
    timelock, qualified-counsel review.
  * **Geo-Restrictions** card (red, live): US + OFAC (CU/IR/KP/SY/RU/BY), HTTP 451.
  * **Token-Disclosure Interstitial** card (blue, live): server-rendered, `bscscan.com` allowlist.
- Disclaimer paragraph rewritten to "No active token sale; any previously linked third-party
  sale has been removed pending qualified-counsel review."
- Chart.js label arrays: LBP P1/P2/P3 entries removed; chart now shows the 7-bucket distribution
  without any LBP labels.

### 10.4 Homepage — Security Status section (commit `4dafcd9`, deploy `v e4e9561a`)

`web/app/[locale]/page.tsx` — new section between the trust strip and the closing CTA:

- Heading: "The path to mainnet is gated by audit, not by a token sale."
- Four green gate tiles (Mythril, Slither, 100% Coverage, 79+ Tests) — each enforced in CI on
  every commit to `main`.
- One amber "Pending — owner action" panel listing the three manual gates (audit, multisig +
  timelock, counsel review).
- Two live compliance cards: Geo-block (HTTP 451) and Token-disclosure interstitial.
- Outbound link to the full `SECURITY-AUDIT.md` in `devswap-docs` (public).

i18n: +20 `home.security*` keys EN + AR (parity 514/514, legal-language CI green).

### 10.5 Critical-tool informed-consent banner (commit `4e90b49`, deploy `v 69b503b7`)

`web/app/[locale]/post/page.tsx` — banner shown ABOVE the funding form, BEFORE the wallet
connect gate. The geo-block (§9.3) is the back-stop for US + OFAC; this banner is the
proactive consent surface everyone else sees before transferring USDT into the smart contract.

Body uses "the smart contract — not the company — locks the funds and releases them only on
your approval" — `locks` (not `holds`/`نحفظ`) is the legal-language CI-compliant verb for the
smart-contract-as-actor framing (CLAUDE.md §18).

i18n: +3 `post.disclaimer*` keys EN + AR (parity 517/517).

### 10.6 Mythril hard-gate — owner action reminder

§9.4 wired the workflow. The remaining piece is the GitHub branch-protection setting which
makes `mythril (P5 hard gate) / mythril` a **required status check** on `main`. Until that
toggle is set, the gate runs and reports, but a merge could theoretically proceed without it
passing. The toggle is at: `https://github.com/DevSwap-org/DevSwap/settings/branches → main →
Require status checks to pass before merging → add 'mythril (P5 hard gate) / mythril'`.

### 10.7 Dangling-reference status (verified post-sync)

After all cycles above, every remaining substring match for `LBP`, `Fjord`, or `token sale` in
the codebase is **explanatory** — text inside the new sections that explicitly states the LBP
was cancelled (e.g., "The previously documented Fjord LBP has been cancelled"). No promotional
or directional reference to the cancelled offering remains in any user-facing surface.

The `STATE.md` and `TASKS.md` journals deliberately preserve the historical pre-pivot entries
(strikethrough) so future readers can trace the decision lineage — these are audit-trail
records, not active links.

### 10.8 Brand identity — "Claude" mentions

Owner directive included "replace Claude with DevSwap Protocol everywhere". Comprehensive scan
across the user-facing surface (web/app, web/components, web/messages, public HTML, README,
docs, contracts, deployed assets) returned **zero matches** in any rendered user-facing string.

The matches that DO exist are confined to internal tooling that **must not be renamed** without
breaking the project:

- `CLAUDE.md` (root) — the Claude Code harness configuration file. Renaming breaks the tool's
  startup contract; out of scope.
- `.claude/` directory + `.claude/OPERATING_MANUAL.md` — Claude Code's reserved namespace for
  sub-agent personas + persisted plans.
- Git commit `Co-Authored-By:` trailer — a convention for AI-assisted code provenance. Future
  commits in this session drop the trailer per the owner's "independent corporate identity"
  directive; historical commits are not rewritten (force-push to `main` would be destructive
  per CLAUDE.md §2.9).
- `cloudflare/README.md` — internal setup README mentioning the Claude shell used to mint the
  CF API token; not user-facing.
- `assets/blockchains/*` — Trust Wallet asset registry mirror (third-party content, unrelated
  to DevSwap's product surface).

Conclusion: the public-facing product identity is already free of "Claude" branding. Internal
tooling preserves it for technical-correctness reasons documented above.
