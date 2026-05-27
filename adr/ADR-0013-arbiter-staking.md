# ADR-0013 — Arbiter Staking Pool with weighted random draw + slashing

- **Status:** Proposed (architectural spec — implementation in V2.3+ via separate ADR-of-implementation)
- **Date:** 2026-05-26
- **Deciders:** Owner + Lead Blockchain Architect
- **Relates to:** ADR-0003 (V2.1 arbiter hardening — `arbiterSince` snapshot), the protocol's
  internal dispute-resolution roadmap (A5/A6 staked random pool), and
  [GOVERNANCE.md §7](../GOVERNANCE.md) which reserves the policy of arbiter selection to DAO Tier-2.
- **Supersedes (when implemented):** the V2.1/V2.2 single-arbiter-per-dispute resolution path.

---

## Context

The V2.1/V2.2 dispute model (ADR-0003) closed the puppet-arbiter attack class via the
`arbiterSince[caller] <= disputeRaisedAt` snapshot rule, but it left two unresolved weaknesses:

1. **Centralized appointment.** Arbiters are added to the registry by the owner (or, in G3+, by a
   DAO vote on individual addresses). At low scale this is acceptable; at protocol-product–market
   fit (≥ 100 disputes/month) it (a) bottlenecks on a single human's bandwidth, (b) creates a
   social-engineering surface, and (c) does not survive G4 (full decentralization) because no
   single entity may appoint anyone after Guardian sunset.

2. **No skin in the game.** A registered arbiter pays no cost to ignore a dispute or vote
   maliciously. Their only loss is reputational (removed from the registry on appeal), and
   `removeArbiter` is immediate but does not retroactively change the bad ruling. This invites
   cartelization — a small group of registered arbiters could coordinate verdicts across many
   disputes without any economic penalty for misbehavior.

The fix is the **staked random pool** pattern (Kleros, Aragon Court), adapted for our constraints:

- BSC mainnet (no native VRF in the L1; Chainlink VRF v2.5 is the canonical option but adds an
  external dependency we'd like to defer until justified by volume).
- The protocol utility token (`$DSWP`) is capped at 100M and is the natural stake/slash asset.
- The escrow already charges `DISPUTE_DEPOSIT = 5 USDT` from the raiser (ADR-0012/V2.2). That
  deposit is the right size for arbiter compensation — no new fee on disputants is needed.

This ADR specifies the on-chain mechanics. Legal framing (whether staking constitutes a security)
and parameter tuning (stake floors, voting windows) are deferred to subsequent ADRs.

---

## Decision

Build `ArbiterStakingPool.sol` as a standalone contract that the V2.3+ escrow consults at
`raiseDispute` time. Active arbiter membership is *derived from stake*, not appointed. Resolution
is by majority vote of a randomly drawn 3- or 5-arbiter panel; the loser's USDT deposit pays the
majority; the minority and any non-voter has their `$DSWP` stake partially slashed.

### 1. Staking & registry

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `MIN_ARBITER_STAKE` | **50,000 DSWP** | 0.05% of `MAX_SUPPLY` — high enough to filter farmers, low enough that early arbiters are not capital-gated out of participating |
| `STAKE_LOCK_DURATION` | **30 days** rolling | An arbiter who calls `unstake()` enters a 30-day cooldown before the token is released; this is the slashing window for any disputes they participated in |
| `MAX_ACTIVE_ARBITERS` | **uncapped, but `MIN_POOL_SIZE = 12`** | Below 12 stakers, drawing 5-arbiter panels skews towards drawing the same individuals repeatedly; below 12 the protocol reverts on `raiseDispute` (the V2.3 escrow falls back to V2.1's single-arbiter path with explicit `LowPool` event) |

Functions:

```solidity
function stake(uint256 amount) external;            // amount >= MIN_ARBITER_STAKE
function topUp(uint256 amount) external;            // increase existing stake (raises selection weight)
function requestUnstake() external;                 // starts the 30-day cooldown; arbiter is REMOVED from eligible draw immediately
function withdraw() external;                       // after cooldown; reverts if any open disputes still implicate the arbiter
```

`arbiterStake[address]` is the per-address stake (zero if not active). `totalStakedWeight` is the
sum used for weighted random draw.

### 2. Random-draw selection (entropy on BSC, no VRF in v1)

**Goal:** select 3 (small disputes) or 5 (large disputes — milestone amount ≥ `LARGE_PANEL_BPS` of
the job total) arbiters from the active pool, weighted by stake, with verifiable randomness that a
single block-producer cannot manipulate within an acceptable cost bound.

**v1 entropy source (BSC-native, no external oracle):**

```solidity
// Combines: block hash of disputeRaisedAt - 1, dispute jobId, milestone index,
//           the disputant's address. Public, post-commitment, and tamper-evident
//           because the disputant cannot retroactively change disputeRaisedAt.
bytes32 seed = keccak256(abi.encode(
    blockhash(disputeRaisedAt - 1),
    jobId,
    milestoneIndex,
    msg.sender,
    address(this)
));
```

**Bias bounds.** A BSC validator can refuse to mine `disputeRaisedAt` to influence
`blockhash(disputeRaisedAt - 1)`, but: (a) the *next* validator would mint the same block hash;
(b) the cost is one block reward (~$0.50 on BSC) per attempt; (c) the validator does not know
in advance which arbiters will be drawn for a given seed because the panel is keccak'd against
all four inputs. The expected gain from biasing one dispute (max 5 USDT deposit) is below the
cost of the attack. Documented as **acceptable for testnet + early mainnet**.

**v2 entropy (post first 10K disputes or post DAO Tier-2 vote):** migrate to **Chainlink VRF v2.5
on BSC** (`0xDA3b641D438362C440Ac5458c57e00a712b66700` subscription manager). This requires the
DAO to fund a subscription with LINK and accept the per-request fee (~0.0005 LINK). The contract
exposes `setVRFConfig(coordinator, subscriptionId, keyHash, requestConfirmations, callbackGas)`
gated by the same governance that controls Tier-2 parameters.

**Weighted draw algorithm** (rejection sampling, O(panel_size × log(active_count))):

```solidity
function _drawPanel(bytes32 seed, uint8 panelSize) internal view returns (address[] memory) {
    require(activeArbiterCount() >= MIN_POOL_SIZE, "LowPool");
    address[] memory panel = new address[](panelSize);
    uint256 cursor = uint256(seed);
    for (uint256 i = 0; i < panelSize; ) {
        // Sample weighted by stake, then de-duplicate (skip if already on the panel)
        address candidate = _weightedPickByStake(cursor);
        bool dup;
        for (uint256 j = 0; j < i; ) { if (panel[j] == candidate) { dup = true; break; } unchecked { ++j; } }
        if (!dup) { panel[i] = candidate; unchecked { ++i; } }
        cursor = uint256(keccak256(abi.encode(cursor)));  // mix forward
    }
    return panel;
}
```

`_weightedPickByStake` walks a `cumulativeStake[]` array. Insertion / removal updates the array
in O(log n) via a Fenwick tree (BIT). For early scale (< 1,000 arbiters), a linear scan is fine
and the BIT is a v1.1 optimization.

### 3. Voting window

- `VOTING_WINDOW` = **7 days** from `raiseDispute`. Any arbiter on the drawn panel must call
  `castVote(jobId, index, VoteSide.Client | VoteSide.Developer)` within the window.
- After the window closes, anyone may call `tallyAndResolve(jobId, index)` (permissionless,
  similar to V2.2's `timeoutDispute`).

### 4. Loser-pays compensation

When `tallyAndResolve` runs:

- Identify the **majority** (≥ ceil(panelSize/2) votes one way). On a tie (only possible on
  the 5-arbiter panel with 2-2-1 abstain → falls to the 3-2 majority by ignoring abstainers;
  on a 3-arbiter panel ties are impossible since panel size is odd and abstain reduces effective
  panel to 2 — which forces a tie-breaker rule: tie → **client wins** by default, the more
  conservative outcome for the funds-holding party).

- **Loser's USDT deposit** (`DISPUTE_DEPOSIT = 5 USDT` per ADR-0012) is split among the **majority
  voters only** (not the minority, not non-voters). On a 3-arbiter panel with a 3-0 verdict,
  each majority arbiter receives `5e18 / 3` USDT. On a 5-arbiter 3-2 verdict, each majority
  arbiter receives `5e18 / 3` USDT (the two minority voters receive nothing but are NOT slashed
  — see §5).

- The job's USDT amount flows to the winning party (client or developer) by the V2.2 milestone
  payout path (no change from existing CEI).

### 5. Slashing

`SLASH_BPS` (default: **500 = 5%** of the arbiter's locked stake per offense).

| Offense | Penalty | Destination of slashed `$DSWP` |
|---------|---------|--------------------------------|
| Non-vote within `VOTING_WINDOW` | `SLASH_BPS` of stake | `burn()` — permanently reduces supply |
| Minority vote on the resolved dispute | **0** — voting in good faith with the minority is not malicious | n/a |
| Appeal overturns the panel's verdict (see §7) | `SLASH_BPS × 2` per majority voter | `burn()` |
| Caught colluding (multi-sig signature off-chain proof, submitted by DAO Tier-2 vote) | Full stake | `burn()` |

The burned token destination is the canonical `burn()` function on `DevSwapToken` (per ADR-0002's
"never `address(0)`" invariant; OZ v5 rejects `address(0)`).

### 6. Invariants

1. **No bypass of the random draw.** Once `raiseDispute` flags a milestone `Disputed`, neither the
   protocol operator, the DAO, nor any individual can substitute or insert arbiters for that
   specific dispute. The panel is locked at `raiseDispute` block; the only state change permitted
   on the panel array is `castVote` (per panelist) and `tallyAndResolve` (terminal).

2. **No retroactive arbiter set changes.** Adding an arbiter (`stake()`) only takes effect for
   disputes raised in **subsequent blocks**. Removing an arbiter (`requestUnstake() → withdraw()`)
   does not affect disputes raised before the removal block — they retain their panel.

3. **Stake conservation.** `sum(arbiterStake[a] for all a) + totalSlashedThisEpoch = stakingPool
   contract's `$DSWP` balance`. Enforced as an invariant test in the test suite.

4. **No double-vote.** `castVote` reverts on second call from the same arbiter on the same dispute.

### 7. Appeal path (deferred to v2)

A losing party may, within `APPEAL_WINDOW = 7 days` of `tallyAndResolve`, post an `APPEAL_BOND =
5 × DISPUTE_DEPOSIT = 25 USDT` and trigger a fresh draw of a 7-arbiter panel. If the 7-arbiter
panel reverses the verdict, the original 3/5-arbiter majority is slashed at `2 × SLASH_BPS`. If
the appeal upholds, the appellant forfeits the bond to the 7-arbiter panel majority.

This adds complexity; out of scope for the V2.3 initial implementation. Documented here so the
state machine reserves the `Appealed` status code.

### 8. Integration with V2.3 escrow

`DevSwapEscrowV2_3.raiseDispute` flow:

1. Existing CEI checks (V2.2).
2. **NEW:** `IArbiterStakingPool(pool).requestPanel(jobId, milestoneIndex, msg.sender, panelSize)`
   returns the drawn panel; the escrow stores it under `panel[jobId][index]`.
3. Emit `DisputePanelDrawn(jobId, index, address[] panel, panelSize, blockhash)`.

`DevSwapEscrowV2_3.resolveDispute` is **removed** for panel-drawn disputes; the function reverts
with `UseTallyAndResolve()`. The owner / single-arbiter path is preserved ONLY for the
`LowPool` fallback (< 12 active stakers).

### 9. Acceptance criteria (for the implementation ADR that follows this one)

- [ ] `ArbiterStakingPool.sol` deployed; `stake`/`requestUnstake`/`withdraw` round-trip green.
- [ ] `_drawPanel` returns 3 unique addresses with stake-weighted distribution (statistical test
      over 10,000 simulated draws: chi-squared p > 0.05 against the expected weighted distribution).
- [ ] `castVote` + `tallyAndResolve` flow tested under 4 scenarios:
      (a) 3-0 unanimous, (b) 2-1 split, (c) 1-1-1 with 1 non-voter, (d) all panelists non-vote.
- [ ] Slashing arithmetic preserves the §6.3 stake-conservation invariant in 10,000 fuzz runs.
- [ ] BSC mainnet-fork test: deploy pool with 15 stakers, run 100 simulated disputes; verify
      drawn-arbiter distribution matches stake weights within 2σ.
- [ ] Slither + Mythril pass (no high; medium FPs documented in SECURITY-AUDIT.md).
- [ ] Gas: `raiseDispute` (with `requestPanel`) ≤ 250,000 gas at panel size 5.

---

## Consequences

**Positive.**

- Eliminates the centralization residual on arbiter appointment.
- Aligns arbiter compensation with dispute volume (no salary, no manual disbursement).
- Burns `$DSWP` on every slash — directly aligned with the existing buyback-and-burn deflationary
  thesis (ADR-0002).
- Survives G4 Guardian sunset: the contract draws panels with zero human input.

**Negative / accepted.**

- Cold-start: requires ≥ 12 stakers locking 50,000 DSWP each = 600,000 DSWP locked before the
  pool activates. Bootstrapping plan: the DAO treasury may seed the initial 12 by recruiting from
  existing OG `$DSWP` holders (TBD in the launch ADR).

- BSC-native randomness has a small validator-bribery surface (§2). Mitigated by the deposit cap
  (5 USDT per dispute) being below the attack cost. Migration to Chainlink VRF is a one-time
  swap behind a `setVRFConfig` admin function (DAO Tier-2 gated).

- Initial implementation adds ~14,000 bytes of bytecode (Fenwick tree + storage layouts). Below
  the EIP-170 limit but eats into headroom for future features.

**Out of scope.**

- The appeal mechanism (§7) — reserved for a follow-up ADR after launch data.
- DAO governance over `MIN_ARBITER_STAKE`, `SLASH_BPS`, `VOTING_WINDOW` — these become Tier-1
  governance parameters per [GOVERNANCE.md §5.1](../GOVERNANCE.md).
- Legal classification of staking (security / not security) — to be reviewed by qualified counsel
  before mainnet activation, per ROADMAP §1.4.

---

## References

- Kleros Court whitepaper (mechanism prior art).
- Aragon Court v1 specification (slashing + appeal patterns).
- Chainlink VRF v2.5 BSC subscription docs (entropy upgrade path).
- ADR-0003 (this ADR's `arbiterSince` snapshot rule is preserved end-to-end as a v2.3 invariant).
- SECURITY-AUDIT.md §6 (V2.1 dispute hardening) — the floor this ADR builds on.
