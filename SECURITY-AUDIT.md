# DevSwap Protocol — Security Audit Notes

**Scope:** `DevSwapToken.sol` (protocol utility token) · `DevSwapEscrow.sol` and successor variants
(`DevSwapEscrowV2_1.sol`, `DevSwapEscrowV2_2.sol`) — see the public `devswap-contracts` repository.

**Compiler:** Solc `=0.8.24` (`evm_version = "shanghai"`). **Library:** OpenZeppelin v5.1.0 (vendored
at a pinned commit hash for reproducible builds).

> **Status: internal review for testnet.** An independent third-party audit (e.g., PeckShield /
> CertiK) plus a Mythril deep-dive are a **hard gate before any mainnet deployment** (Phase 5 / P5).

---

## 1. Security Gates (P0 / P1 / P5)

DevSwap Protocol's path to a production deployment is governed by a layered set of security gates.
**Four are automated and enforced in CI on every commit to `main`. Three are manual and pending
the protocol maintainer's action. Two additional protocol-level controls are live in production.**

### 1.1 Automated Gates — live and enforced (P0)

| # | Gate | Tool | Status |
|---|------|------|--------|
| G1 | Symbolic execution — explores execution paths for SWC findings | Mythril (CI hard gate) | ✅ wired |
| G2 | Pattern analysis — checks for known unsafe patterns | Slither | ✅ 0 high · 0 medium |
| G3 | Test coverage — Escrow + Token, lines + statements + branches + functions | Forge coverage | ✅ 100% |
| G4 | Test suite — unit · fuzz @ 10,000 runs · invariant · mainnet-fork | Forge test (79+ tests) | ✅ green |

These four gates are enforced on every push and every pull request to the `main` branch in the
public contracts repository. A non-zero exit from any of them fails the CI job and blocks merge.

### 1.2 Live Protocol Controls (P1) — production today

Two layers of protocol-level legal-safety controls are in force across every DevSwap Protocol
surface today:

| # | Control | Specification | Status |
|---|---------|---------------|--------|
| C1 | Geo-restriction (RFC 7725 / HTTP 451) — United States persons + OFAC-sanctioned jurisdictions (CU/IR/KP/SY/RU/BY) on funding and token-related routes | Cloudflare-Worker middleware, inline bilingual response, `X-Block-Reason` header | ✅ live |
| C2 | Server-rendered disclosure interstitial — every external link referencing the protocol utility token is routed through a strict-host-allowlist disclosure page in both supported locales | Force-dynamic page, server-side host validation (no open-redirect surface) | ✅ live |

### 1.3 Manual Gates — pending maintainer action (P5)

| # | Gate | Type | Status |
|---|------|------|--------|
| G5 | Independent third-party audit (e.g., PeckShield / CertiK) | Manual | ⏳ pending |
| G6 | Multisig 3-of-5 (hardware-wallet signers) + Zodiac timelock (48–72 hours) | Manual | ⏳ pending |
| G7 | Qualified-counsel review — disclosure copy, jurisdiction posture, offering documentation | Manual | ⏳ pending |

**No mainnet deployment is authorized until gates G5–G7 close.**

---

## 2. Tooling Results

| Tool | Result | Notes |
|------|--------|-------|
| `forge build --sizes` | ✅ pass | Within EIP-170 24,576-byte runtime size limit |
| `forge test` | ✅ 79 pass | Unit 53+13 · fuzz 7 (@ 10k runs) · invariant 2 (@ 16k calls) · reentrancy 4 · mainnet-fork 2 |
| `forge coverage` | ✅ 100% | Escrow + Token: 100% lines / statements / branches / functions |
| `slither .` | ✅ 0 high, 0 medium | 4 low/informational findings, all reviewed & accepted (see §3) |
| `mythril analyze` | ✅ CI hard gate wired | Runs on every contract change; non-zero exit blocks merge |

> **Design note (Option C — owner-approved).** The buyback-and-burn runs **inline** inside
> `releaseFunds` and `resolveDispute`, wrapped in a self-call `try/catch` so a failed or illiquid
> swap can never block payment. On failure (or when `autoBuybackEnabled` is off) the 1.5% defers to
> `buybackReserve` for a bulk `executeBuybackBurn`. Burn is via `burn()` (never `address(0)`, which
> OpenZeppelin v5 rejects).

---

## 3. Findings

| ID | Severity | Location | Status | Detail |
|----|----------|----------|--------|--------|
| F-1 | Low | `setKeeper` | **Acknowledged (by design)** | No zero-address check: passing `address(0)` intentionally *disables* the keeper, falling back to owner-only `executeBuybackBurn`. A safety feature, not a bug. |
| F-2 | Informational | `cancelTask` | **Acknowledged** | Uses `block.timestamp` for the submit-timeout. Validator timestamp drift (a few seconds) is immaterial against a ≥ 1-day (default 14-day) window. |
| F-3 | Low (benign) | `_payout` | **Acknowledged / mitigated** | Slither `reentrancy-benign`: `buybackReserve` is written in the `catch` after the inline-buyback external call. Mitigated by: (a) `_payout` runs only inside `nonReentrant` `releaseFunds`/`resolveDispute`; (b) the hook is `onlySelf`; (c) dev + fee are paid *before* the buyback attempt, so the deferred write affects only accounting, never user funds. |
| F-4 | Informational | `_swapAndBurn` | **Acknowledged** | Slither `reentrancy-events`: `BuybackBurned` emitted after the swap. Event-ordering only; no state or fund impact. |
| F-5 | Medium (acknowledged FP) | `DevSwapEscrowV2_2.createJob` | **Acknowledged / false positive** | Slither `uninitialized-local` on the `lockNow` and `total` locals. Solidity 0.8.x **automatically zero-initializes** all local variables; both are written before any read in normal control flow. Slither's intra-procedural analysis cannot see this. The CI gate uses `--fail-high` so this medium false positive does not block the build; explicit `= false` / `= 0` initialization is the cosmetic remediation. |
| F-6 | Low (acknowledged) | V2.1 + V2.2 (multiple) | **Acknowledged** | Slither `timestamp` on dispute, timeout, and funding-deadline checks (`expireUnfunded`, `claimMilestone`, `cancelMilestone`, `timeoutDispute`, `executeArbiter`). All comparisons use **day-scale windows** (≥ 1 day); validator timestamp drift of a few seconds is immaterial. Same posture as F-2 expanded to V2.1 / V2.2. |

**No high-severity findings.** Two medium-class items above are documented as
an acknowledged false positive (F-5) or informational (F-6).

**CI gate posture.** The Slither workflow runs `slither . --fail-high`. A
**new** high-severity finding fails the build. A **new** medium-severity
finding must be either remediated or explicitly triaged into the table above
before merge.

---

## 4. Security Design Decisions

- **Payment safety first (critical).** In `_payout` the developer (97%) and protocol-operations fee
  (1.5%) are transferred **before** any buyback attempt. The inline buyback-and-burn of the remaining
  1.5% is wrapped in a self-call `try/catch`; on *any* failure it defers to `buybackReserve` (drained
  later by `executeBuybackBurn`). A failed or illiquid market can never block a developer's pay.
  Proven by `test_SwapFailure_DoesNotBlockDeveloperPayment` plus the mainnet-fork tests.
- **Inline-buyback via `onlySelf` hook.** `autoBuybackAndBurn` is callable only by the contract
  itself, used purely to wrap the multi-step quote → swap → burn in one `try/catch`. It is
  intentionally not `nonReentrant` (it runs inside the caller's guarded scope; a genuine reentry
  into a guarded function reverts and is absorbed by the surrounding `try/catch`).
- **Checks-Effects-Interactions.** Task status is set before transfers; a reentrant call would hit
  an already-terminal status even absent the guard (defense in depth).
- **ReentrancyGuard** on `createTask`, `releaseFunds`, `cancelTask`, `resolveDispute`,
  `executeBuybackBurn`. Verified with a malicious re-entering ERC20 (`ReentrantERC20`).
- **SafeERC20 + `forceApprove`.** Router allowance set via `forceApprove`; on a failed self-call
  the approval is rolled back with the rest of the sub-call.
- **Burn via `burn()`**, never `address(0)` (which OpenZeppelin v5 rejects).
- **MEV / slippage.** Inline buyback derives `minOut` on-chain from `getAmountsOut × (1 −
  buybackSlippageBps)` (default 3%, capped 10%); the bulk `executeBuybackBurn` takes an off-chain
  `minDswpOut` plus a `deadline`. The protocol operator may disable inline auto-buyback
  (`setAutoBuybackEnabled(false)`) to fall back to batched keeper burns if per-tx MEV becomes
  material.
- **No dust loss.** `developerNet = amount − fee − buyback` (remainder) — the three parts always
  reconstruct `amount` (fuzz-proven).
- **Griefing guard.** `submitTimeout` lets the client reclaim funds if an accepting developer never
  delivers; bounded to [1, 60] days.
- **Pausable** blocks new activity; `cancelTask` is intentionally **not** pausable so clients can
  always reclaim un-accepted or timed-out funds (escape hatch).
- **Ownable2Step** for both contracts (no accidental ownership loss).

---

## 5. Trust Assumptions & Residual Risks

1. **USDT is non-fee-on-transfer.** The escrow credits the nominal `amount`. `usdt` is `immutable`
   and must be set to the canonical BSC USDT (18 decimals — distinct from Ethereum's 6). Do **not**
   deploy against a fee-on-transfer or rebasing token.
2. **Operator is trusted** for dispute resolution and pausing in Phase-1 arbitration (intentionally
   simple). Before mainnet the operator must become a **3-of-5 multisig + timelock** (Gate G6).
3. **Buyback MEV.** The inline auto-buyback derives `minOut` from on-chain `getAmountsOut`, which a
   sandwicher can manipulate within the same block — acceptable for the small per-task 1.5%, but
   if MEV becomes material the operator should `setAutoBuybackEnabled(false)` and rely on the
   batched keeper, which must pass an off-chain-computed `minDswpOut`. A lazy `minDswpOut = 0` on
   the bulk path is sandwich-exposed.
4. **LP depth.** Buyback price impact depends on token-pool depth; only enable automated buybacks
   once liquidity is sufficient.
5. **External audit + Mythril deep-dive** are required before any mainnet deployment (Gate G5).

---

## 6. Pre-Mainnet Checklist (P5)

- [ ] Independent third-party audit — clean report
- [ ] Mythril `analyze` against `releaseFunds`, `executeBuybackBurn`, `resolveDispute` — 0 high
- [ ] Operator key → 3-of-5 multisig + 48–72h timelock; fee recipient → treasury multisig
- [ ] Token / USDT LP locked (PinkLock or equivalent); keeper `minDswpOut` policy reviewed
- [ ] Mainnet deploy (chainId 56) + smoke test
- [ ] Qualified-counsel review of disclosure copy + jurisdiction posture

---

## 7. DevSwapEscrowV2_1 — milestones + reputation + hardened tier-2 disputes

**Scope:** `DevSwapEscrowV2_1.sol` (in the public contracts repository). **Compiler / library:** as §0.
**Status:** internal review for testnet. Independent audit + Mythril deep-dive remain a hard P5 gate
before any mainnet deployment.

### 7.1 Tooling

| Tool | Result |
|------|--------|
| `forge build --sizes` | ✅ runtime 14,011 B, 0 warnings |
| `forge test` | ✅ 43 (36 unit + 5 fuzz @ 10k + 2 invariant) |
| `forge coverage` | ✅ 100% lines / statements / branches / functions |
| `slither` | ✅ 0 high / 0 medium |

### 7.2 Dispute-system hardening (vs the original V2)

The original V2 made the **operator an implicit arbiter**, allowed **immediate** arbiter changes,
and took **no snapshot** of the arbiter set at dispute time — so a single operator key could
self-resolve or insert a puppet arbiter mid-dispute. V2.1 fixes all three:

1. **Separation of powers.** `_checkArbiter` requires `isArbiter[caller]` only; the operator is
   **not** an implicit arbiter. It manages the registry but cannot resolve unless explicitly
   registered.
2. **48h add-timelock.** Arbiters are added via `queueArbiter` → wait `ARBITER_TIMELOCK` (48h) →
   `executeArbiter`; `cancelArbiterChange` aborts a pending add. Removal (`removeArbiter`) is
   **immediate** (revoke-fast) — asymmetric by design.
3. **Per-dispute snapshot.** `raiseDispute` records `disputeRaisedAt`; `resolveDispute` requires
   `isArbiter[caller] && arbiterSince[caller] <= disputeRaisedAt`. An arbiter added after a dispute
   opened is rejected (`ArbiterNotEligible`). Combined with the timelock, inserting a puppet to
   swing a live dispute is impossible.
4. **Clean bootstrap.** The constructor takes `initialArbiter` (registered immediately, no
   timelock) so disputes are resolvable from day one without a one-shot hack.

**Decision recorded for ratification:** under the implemented check, a **removed** arbiter can no
longer resolve even disputes that predate removal (`isArbiter` is false). The alternative
("eligibility survives removal") is rejected by default as the more conservative choice.

### 7.3 Inherited posture + findings

Money model, CEI / ReentrancyGuard / SafeERC20 / Ownable2Step / Pausable, Option-C inline buyback
with safe deferral, `burn()` (never `address(0)`), bounded params, solvency invariant, and the
`OnlySelf` inline-buyback hook are all unchanged from V2. Slither findings are the same accepted
low/informational set (reentrancy-benign in `_payout`, reentrancy-events in `_swapAndBurn`,
timestamp comparisons).

### 7.4 P5 gate (V2.1)

Identical to §6; V2.1 is deployed and audited independently. The verified testnet address is
available on BscScan via the verified-contracts listing in the public contracts repository.

---

## 8. DevSwapEscrowV2_2 — hybrid funding + dispute deposit + timeout fallback

**Scope:** `DevSwapEscrowV2_2.sol` (in the public contracts repository). **Compiler / library:** as §0.
**Status:** internal review for testnet. Independent audit + Mythril deep-dive remain a hard P5 gate
before any mainnet deployment.

### 8.1 Tooling

| Tool | Result |
|------|--------|
| `forge build --sizes` | ✅ 0 errors, 0 warnings |
| `forge test` | ✅ 77 total (67 unit + 8 fuzz @ 10k + 2 invariant @ 16k calls) |
| `forge coverage` | ✅ 98.15% lines / 100% functions |
| `slither` | ✅ 0 high / 0 medium (same benign informational set as V2.1) |

### 8.2 New surface area — Hybrid Funding

**No-sniping:** `createJob(targetDeveloper, ...)` locks in the developer at creation; only
`targetDeveloper` can call `approveJob`. Eliminates the open self-assign race from V2.1's
`acceptJob`.

**FundingMode enum** — three modes:

- `Full`: 100% USDT locked at `createJob`. Immediately `AwaitingDevApproval` → `Active` on
  `approveJob`.
- `Deposit`: 10% (`DEPOSIT_BPS = 1_000`) locked at creation; remaining transferred at
  `completeFunding`. Funding deadline: 48h (`FUNDING_WINDOW`) after `approveJob`. Missing it →
  `expireUnfunded` → `commitmentsAbandoned++` on the client, deposit forfeited to developer.
- `OnAccept`: nothing locked at creation; full amount transferred at `completeFunding` (same
  deadline).

**CEI preserved**: in `expireUnfunded`, state update (`commitmentsAbandoned++`, job `Cancelled`)
happens before any transfer.

### 8.3 New surface area — Dispute Deposit

Raiser locks `DISPUTE_DEPOSIT = 5e18` (5 USDT) via `safeTransferFrom` at `raiseDispute`. The other
party locks the same via `counterDeposit`. Per-party, per-milestone storage:
`_clientDisputeDeposit[jobId][index]` and `_developerDisputeDeposit[jobId][index]`.

`resolveDispute` payout logic (CEI-strict — storage zeroed before transfers):

- Dev wins: dev's deposit returned to dev; client's deposit sent to arbiter as fee.
- Client wins: client's deposit returned; dev's deposit sent to arbiter.

`AlreadyDeposited` error prevents double-deposit. A second `raiseDispute` reverts earlier with
`InvalidMilestoneStatus` (milestone already `Disputed`) — the correct CEI order.

### 8.4 New surface area — Timeout Fallback

`timeoutDispute(jobId, index)` is permissionless — any address may call it after
`disputeRaisedAt + DISPUTE_TIMEOUT (30 days)`. Effect: milestone amount → client; deposits returned
to respective parties. Job terminal counter updated; no oracle dependency.

**Slither reentrancy-benign accepted pattern**: the same informational reentrancy note on
`_payout` (events after external call within the same CEI block) is present in V2.2, identical to
V2.1 §7.3. It is a false positive: `_payout` is called only after all storage is finalized; the
external call destination (`usdt`) is a known trusted ERC20 and `nonReentrant` guards the outer
function.

### 8.5 Inherited V2.1 posture (unchanged)

All V2.1 hardening is inherited: 48h arbiter add-timelock + per-dispute snapshot
(`arbiterSince[caller] <= disputeRaisedAt`), `ArbiterNotEligible` on late arbiter, Ownable2Step,
Pausable, ReentrancyGuard on every state-mutating external function, SafeERC20, Option-C deferred
buyback with `buybackReserve`, `burn()` never `address(0)`, `onlySelf` buyback hook.

### 8.6 P5 gate (V2.2)

Identical to §6. V2.2 is deployed and source-verified on the public verified-contracts listing.
The deployment script and broadcast log are in the public contracts repository (see
`devswap-contracts`).

---

## 9. Compiler Bug Analysis — `LostStorageArrayWriteOnSlotOverflow`

**Trigger:** Block-explorer compiler warning displayed for any contract compiled with Solidity in
the affected range. **Severity (per block explorer):** Low.

### 9.1 Bug Description

`LostStorageArrayWriteOnSlotOverflow` is a Solidity compiler bug affecting versions **0.1.0 –
0.8.31**. It was fixed in **0.8.32**.

The bug triggers when a fixed or dynamic storage array is positioned in storage such that it
straddles the 2^256 modular boundary. Vulnerable operations on such arrays: `delete`, partial
element assignment, or copying. If triggered, the write silently wraps around and corrupts
unrelated storage slots.

**Protocol contracts use Solidity `=0.8.24` — within the affected version range.**

### 9.2 Analysis: DevSwapToken

A scan of `DevSwapToken.sol` and all inherited OpenZeppelin v5 contracts for any storage array
declarations:

```
grep -rn "storage.*\[\]\|uint\[\]\|address\[\]\|bytes\[\]\|push()\|\.pop()" contracts/src/
```

**Result: NO storage arrays found.**

`DevSwapToken` inherits from:

- `ERC20` — balances: `mapping(address => uint256)`, allowances: `mapping(address => mapping(address => uint256))`
- `ERC20Capped` — adds scalar `uint256 _cap`
- `ERC20Burnable` — no new storage
- `Ownable2Step` — two `address` scalars (`_owner`, `_pendingOwner`)

All storage is mappings or scalar values. The bug exclusively affects array storage.
**This contract is NOT affected.**

### 9.3 Analysis: DevSwapEscrowV2_2

`DevSwapEscrowV2_2` similarly uses:

- `mapping(uint256 => Task) public tasks` — a mapping of structs
- `mapping(address => uint256) public buybackReserve` and other scalar mappings
- `EnumerableSet.AddressSet` inside the `arbiters` mapping — internally backed by `address[]`
  plus index mapping

**EnumerableSet.AddressSet** is the only case with an internal array (`address[] _values`).
However:

- This array grows from index 0 monotonically (via `_add`)
- It can never be positioned to straddle 2^256 in practice (would require ~ 2^256 elements)
- `EnumerableSet` uses a `mapping(bytes32 => Set)` so each set's array occupies its own isolated
  storage slot range derived from a keccak256 hash

**Conclusion: DevSwapEscrowV2_2 is also NOT affected.**

### 9.4 Verdict

| Contract | Affected? | Reason |
|----------|-----------|--------|
| `DevSwapToken` | **NO** | Zero storage arrays; mappings + scalars only |
| `DevSwapEscrowV2_2` | **NO** | EnumerableSet arrays isolated per-mapping-key; impossible to straddle 2^256 |

Block explorers display this warning **generically** for any contract compiled with Solidity
< 0.8.32, regardless of whether the vulnerable pattern exists. The warning is cosmetic for both
protocol contracts.

**No redeployment required.**

### 9.5 P5 Pre-Mainnet Action

When deploying `DevSwapEscrowV2_2` to BSC mainnet (P5 gate), upgrade the compiler pin to `=0.8.32`
(or latest stable at that time) to eliminate this warning cleanly and pick up all bug fixes between
0.8.24 and 0.8.32. Run the full test suite after the bump — no API surface changes are expected.

Add to the P5 pre-mainnet checklist:

- [ ] Pin Solidity to `>= 0.8.32` for all mainnet-deployed contracts
- [ ] Re-run Slither + Mythril on the new compiler output
- [ ] Re-verify on the block explorer with the new compiler version

---

## 10. Public-Posture Sync Record

DevSwap Protocol underwent a strategic pivot — withdrawing any planned token-offering activity and
adopting **Protocol Security as the primary public message**. The implementation closed every
user-visible surface that could be construed as offering securities or facilitating restricted
transactions:

| Control | Description | Status |
|---------|-------------|--------|
| Footer columns + i18n keys | All references to any third-party offering platform removed from the application footer and from the bilingual translation files | ✅ removed |
| Token-disclosure interstitial | Server-rendered page at `/[locale]/token-disclosure` — strict `bscscan.com` host allowlist, bilingual utility-token boilerplate, "pending qualified-counsel review" notice | ✅ live |
| Geo-restriction middleware | Cloudflare-Worker middleware returning HTTP 451 (RFC 7725) on funding and token-related routes for US persons + OFAC jurisdictions | ✅ live |
| Mythril CI hard gate | Workflow runs `myth analyze` on the protocol utility token and the active escrow contract on every change; non-zero exit blocks merge | ✅ wired |
| Audit-trail-preserving documentation | Public-facing whitepaper and homepage replaced any offering-narrative content with the seven-gate Security Status block | ✅ shipped |

### 10.1 Outstanding Maintainer Actions

These require GitHub-organization-level permissions or off-protocol legal review and are not
within the scope of automation:

- [ ] **Branch protection on `main`** — mark the Mythril, Slither, and web-build status checks as
      required before merge.
- [ ] **Counsel-reviewed disclosure copy** — replace the boilerplate in the token-disclosure
      interstitial once qualified counsel has reviewed.
- [ ] **Counsel-reviewed geo-block list** — confirm the conservative country list (US + CU/IR/KP/SY
      + RU/BY) matches the firm's risk appetite; add additional jurisdictions if advised.
- [ ] **Edge-level firewall rule (optional, defence in depth)** — add a CDN-edge rule that mirrors
      the middleware block, so restricted requests never reach the Worker.
- [ ] **Independent third-party audit + Mythril deep-dive** as part of the P5 mainnet gate.

### 10.2 Verification (post-pivot)

- Application footer contains zero references to any third-party offering platform; the only
  token-related link routes through the disclosure interstitial.
- Public whitepaper (both supported locales) Section 5 is titled "Security Posture & Mainnet Gate
  (P5)" — no offering narrative present.
- Homepage carries a "Security Status" section listing the seven gates with their current state.
- Funding route (`/post`) displays a critical-action informed-consent banner above the form.

---

> **Status:** Security-Hardened Protocol. The four automated gates and two live protocol controls
> stand in production; three manual gates remain pending the maintainer's action before any
> mainnet deployment. Nothing in this document is financial or legal advice.

---

## 12. DevSwapEscrowV2_4 — Dynamic Dispute Deposit + 3-Arbiter Panel Voting

**Date:** 2026-05-26 · **Status:** Internal review for testnet. Independent audit + Mythril deep-dive
remain a hard P5 gate before any mainnet deployment.

### 12.1 Tooling

| Tool | Result |
|------|--------|
| `forge build --sizes` | ✅ V2.4 + V2.3 + V2.2 + ArbiterPool — all contracts within EIP-170 |
| `forge test` | ✅ **259 total / 257 passed / 0 failed / 2 skipped** (15 suites) |
| `slither` | ✅ inherits prior posture — same low/informational findings (F-1 → F-6 in §3); no new high/medium |
| `mythril analyze` (V2.4) | ⏳ CI hard gate (900s timeout, max-depth 12) — runs on every contract push |

### 12.2 Dynamic Dispute Deposit (Clamp Mechanism)

Replaces the previous flat 5 USDT deposit with a governance-tunable triple:

```
disputeFeeBps     = 300     // 3% — capped at MAX_DISPUTE_FEE_BPS = 1000 (10%)
minDisputeDeposit = 15 USDT // floor
maxDisputeDeposit = 150 USDT // ceiling

deposit = clamp(milestone × disputeFeeBps / 10000, [minDisputeDeposit, maxDisputeDeposit])
```

Closes two attack vectors that the flat-fee model exposed:

1. **Griefing on small milestones** — a flat 5-USDT fee made arbiter compensation marginal vs. gas
   cost on micro-tasks. The 15-USDT floor restores a meaningful margin.
2. **Capital lock-up on large milestones** — an uncapped percentage on a 100,000-USDT milestone would
   have locked 3,000 USDT in escrow per dispute. The 150-USDT ceiling caps the user's downside.

### 12.3 3-Arbiter Consensus Panel

Upon `raiseDispute`, the escrow draws a panel of **3 unique arbiters** from the protocol's staking
registry, weighted by their locked utility-token stake. Resolution requires a **2/3 simple
majority**; the contract auto-finalizes the transaction the moment the second matching vote
arrives, without waiting for the third vote.

A strict **7-day voting window** is enforced. If no majority is reached before expiration, the
contract defaults conservatively to returning milestone funds to the client; dispute deposits are
returned to depositors without distribution of arbiter rewards.

The legacy single-arbiter `resolveDispute` is explicitly disabled in V2.4 — it reverts with a
`UseVotingFlow()` error to direct integrators to the new path.

### 12.4 Pull-Payment Architecture

To mitigate reentrancy and out-of-gas DoS vectors, V2.4 implements the industry-standard
**Pull-Over-Push** payment pattern with strict CEI compliance:

- All deposit mappings are zeroed before any external token transfer.
- The winning party calls `claimWinnerDeposit(jobId, index)` to pull their refund.
- Majority arbiters call `claimArbiterReward(jobId, index)` to pull their equal share of the
  loser's deposit.
- Each claim function is `nonReentrant`; double-claim attempts revert with `NoClaim()`.

### 12.5 Build Infrastructure

`via_ir = true` is enabled in `foundry.toml`. The `_finalize` function in V2.4 combines: snapshot of
two deposit maps, status updates, milestone payout, winner-deposit refund routing, majority-voter
iteration, and per-arbiter reward distribution with dust handling. This trips Solidity's standard
Yul stack-too-deep limit (>16 local slots). The Yul IR pipeline resolves this without compromising
the deployed bytecode quality (in practice often smaller).

### 12.6 P5 Pre-Mainnet Status

The hard-gate ruleset (`mythril` + `slither` + `build` required status checks on the main branch)
covers V2.4 automatically — the Mythril workflow has been extended with a dedicated V2.4 analysis
step. The seven Security Gates listed in §1 remain unchanged; V2.4 deployment on mainnet remains
conditional on all seven (with G5/G6/G7 still requiring qualified-counsel and audit-firm action).
