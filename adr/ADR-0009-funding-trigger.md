# ADR-0009 — Funding model: client-selected commitment mode (hybrid: Full / 10% deposit / On-accept)

- **Status:** **Accepted** — owner chose the hybrid 3-mode model (2026-05-24, «ايه خليها هجين … اختر أفضل
  الممارسات»). Two operational sub-decisions are flagged **tunable** below (recommended defaults given).
  Contract code is **not yet written** — implementation is the next stream (TDD), this ADR is its spec.
- **Date:** 2026-05-24 (supersedes the Proposed A/B/C draft of this ADR)
- **Relates:** ADR-0007 (converge on V2.1 — refines its *funding trigger*), ADR-0002 (milestone-only —
  carried forward, no hourly), ADR-0003 (arbiter), DOC-4 Stage 4, DOC-5 §3/§5, DOC-6 §C.1/§D.3.2;
  CLAUDE.md §17/§18 + project rules (security-first, TDD).
- **Trigger:** owner observation — *«ما أعتقد أن أحداً يريد أن يقفل أمواله إلى أن يجيه مطوّر»* → owner
  decision to let the **client choose** how much to lock at post time.

## Context (grounded in the live contract)

Verified in [`DevSwapEscrowV2_1.sol`](../../contracts/src/DevSwapEscrowV2_1.sol): `createJob`
([:237](../../contracts/src/DevSwapEscrowV2_1.sol:237)) pulls the **full total** at creation into status
`Open` with **no developer**; `acceptJob` ([:240–249](../../contracts/src/DevSwapEscrowV2_1.sol:240)) lets
**any non-client** self-assign (moves no funds). This produced **three defects** (the A/B/C analysis that
preceded this decision): (1) **capital dead-lock** — funds frozen before any counterparty commits; (2)
**Gig sniping** — any developer can capture a funded job (no `targetDeveloper` bind); (3) matches **neither**
incumbent — Fiverr = pre-committed Gig (`fiverr.md:2352`, `fiverr.md:2325`), Upwork = fund-**after**-accept
(`upwork.md:476`, `upwork.md:488-500`).

**Owner's resolution (chosen):** don't pick one funding trigger — make it **hybrid**: the client who posts
chooses one of **three funding modes**, each cancelable in full **before the developer approves**:

1. **Full lock now** — locks 100% at post. Strongest trust signal → fastest developer acceptance.
2. **10% good-faith deposit** — locks 10% at post. A seriousness signal without tying up full capital.
3. **On-accept** — locks nothing until the developer approves. Lowest friction for the client, **lowest
   trust** → proposals may be slower or ignored.

All three: **the client can cancel and recover the locked amount at any time before developer approval.**

## Design (the chosen model)

Two **orthogonal axes**. The owner's three options are Axis B; they apply to **both** doors of Axis A.

- **Axis A — the door (who the developer is):**
  - **Gig door** — developer = the Gig's owner (terms pre-committed by listing the Gig).
  - **Open-Job door** — client posts free, developers propose free, **client selects** one proposal.
  - In **neither** door does a developer self-assign by racing — acceptance is always the **intended**
    developer. *(This kills defect #2 — sniping — for every mode.)*
- **Axis B — the funding mode (the owner's 3-way choice):** Full / Deposit-10% / On-accept.

### Unified lifecycle (door sets the developer; mode sets the funding)

```
POST  (door D, mode M)
  ├─ D=Gig : developer = Gig owner            ├─ M=Full   : lock 100%
  └─ D=Open: developer = TBD (free proposals) ├─ M=Deposit: lock 10%
                                              └─ M=OnAccept: lock 0%
  │   ── client may CANCEL anytime here → return the full locked amount, no penalty ──
  ▼
DEVELOPER APPROVAL  (موافقة المطور)
  ├─ D=Gig : developer confirms the order      (or instant, if owner enables instant-order)
  └─ D=Open: the client-selected developer accepts the offer
  ▼
FUNDING COMPLETION  (invariant: 100% escrowed before any work starts)
  ├─ M=Full    : already 100%        → ACTIVE immediately   (fastest)
  ├─ M=Deposit : client tops up 90%  within the funding window (the 10% counts toward it)
  └─ M=OnAccept: client funds 100%   within the funding window
       │   on top-up/fund FLAKE (window elapses): return the locked amount to the client,
       │   close the job, and increment the client's PUBLIC reputation counter
       │   `commitmentsAbandoned` — seriousness is enforced by REPUTATION, not forfeiture.
  ▼
ACTIVE → milestones → release → instant settlement      (existing V2.1 lifecycle, unchanged)
```

**Invariant preserved:** a developer never works on unfunded escrow — in every mode the job becomes ACTIVE
only once 100% is locked. The modes change *when* the lock completes relative to approval; they never let
work start unfunded. *(This resolves defect #1 — the client only locks early if they choose to, and can
always cancel before a developer commits.)*

## Best practices chosen (the «اختر أفضل الممارسات» part)

1. **Smart default per door.** Gig door defaults to **Full** (it's an instant purchase, Fiverr-style);
   Open-Job door defaults to **Deposit-10%** (serious signal without full lock). Client can override.
   *(tunable — see below.)*
2. **Make the trust signal legible to developers.** Every listing/offer shows a badge — **"Fully funded" /
   "10% deposit locked" / "Funds on accept"** — so developers self-select. This *is* the market mechanism the
   owner is creating, and it leans on the transparency MOAT (DOC-3 W5).
3. **Reputation, not forfeiture, enforces seriousness.** A client who locks then flakes **after** a developer
   commits loses no money (the deposit returns) but gains a **public `commitmentsAbandoned` counter**.
   Rationale: keeps §18 clean (no money moves without delivered work / no penalty-custody semantics) and
   turns DevSwap's on-chain reputation into the enforcement layer (DOC-3 W3). *(forfeiture is the flagged
   alternative below.)*
4. **Cancel-before-approval always returns 100% of the locked amount** — the escape valve that dissolves the
   dead-lock objection. No penalty at this stage (no developer has committed yet).
5. **Bind the developer — no sniping, ever.** Acceptance is by the Gig owner (Gig door) or the
   client-selected developer (Open-Job door); no open self-assign race.
6. **Funding window after approval (modes 2/3).** A bounded window for the client to complete funding;
   expiry auto-closes + records the counter. *(tunable length.)*
7. **Allowance UX.** For modes 2/3 the client `approve`s a USDT allowance up front (approval ≠ transfer); the
   top-up/fund transfer is **client-initiated** at completion (no developer-triggered pull of client funds —
   the safer choice; the funding window + reputation counter handle client inaction).
8. **Plain-language, §18-safe copy** at the mode selector and on the badges (en/ar), routed through
   `design:ux-copy`; `design:accessibility-review` before any UI merge.

## Consequences

- **Client:** full control over capital exposure; never frozen without choosing to, always cancelable before
  a developer commits. Directly answers the owner's objection.
- **Developer:** still never works on unfunded escrow; the funding-mode badge lets them price risk (a Full job
  is safest to accept; an On-accept job is a gamble they can decline).
- **Market dynamic:** funding mode becomes a **credible signal** — serious clients lock more and get faster,
  better proposals; this is healthy two-sided selection, not a hidden ranking.
- **Security (the sensitive surface):** the new transfer paths (deposit lock, cancel-return, top-up,
  expire-return) must each be CEI + `ReentrancyGuard` + `SafeERC20`, exact-amount, with safe 10% math on
  **18-decimal** USDT. Client-initiated funding avoids the developer-triggered-drain risk entirely.
- **Audit surface ↑** (a small state machine: `AwaitingDevApproval`, `AwaitingFunding`) — justified by
  removing real friction + a sniping vector, and **cheap now** (pre-mainnet, testnet-only, no fund migration).
- **Docs to update on implementation:** DOC-4 Stage 4, DOC-5 §3/§5, DOC-6 §C.1/§C.2/§D.3.2 (Fund sheet),
  CONTRACTS.md, ARCHITECTURE.md, SECURITY-AUDIT.md, TASKS.md.

## Implementation plan (next stream — TDD, security-first)

> Pre-mainnet + testnet-only ⇒ **amend `DevSwapEscrowV2_1` and redeploy** (legacy deploy → read-only, no
> fund migration). Mainnet stays gated behind the P5 audit/multisig gate (ADR-0007).

1. **Contract.**
   - `enum FundingMode { Full, Deposit, OnAccept }`; add to the job; `enum` extends `JobStatus` with
     `AwaitingDevApproval`, `AwaitingFunding`.
   - **Gig:** `orderGig(developer, milestoneAmounts[], metadataHash, mode)` — locks per mode, binds the Gig
     developer, status `AwaitingDevApproval` (or `Active` if Full + instant-order enabled).
   - **Open-Job:** `createOffer(developer, milestoneAmounts[], metadataHash, mode)` (client-selected dev,
     locks per mode) → `acceptOffer` (named dev only) → Full ⇒ `Active`; Deposit/OnAccept ⇒ `AwaitingFunding`.
   - `cancelBeforeApproval` (client; any pre-approval state) → return the exact locked amount.
   - `completeFunding` (client; `AwaitingFunding`) → pull remaining to 100% → `Active`.
   - `expireUnfunded` (anyone/keeper after the window) → return locked amount, close, `commitmentsAbandoned++`.
   - Reputation struct gains `commitmentsAbandoned`. Keep all guards + timeouts + events
     (`Ordered/OfferCreated/Approved/FundingCompleted/Expired/Cancelled`).
2. **Tests (≥95% Escrow coverage):** every (door × mode) happy path; cancel-before-approval returns the exact
   locked amount; deposit top-up completes; top-up/fund **flake → expire → return + counter++**; **no-sniping**
   (only the intended dev approves); reentrancy on every transfer path; **fuzz** the 10% rounding (18-dec);
   **invariant** solvency across all modes; double-approval & cancel-after-approval rejected.
3. **Frontend (en/ar, §18-safe; `design:ux-copy` + a11y review):** mode selector at post with trade-off copy;
   developer-facing trust badges in Explore; cancel-before-approval action; top-up flow (modes 2/3);
   reputation display incl. `commitmentsAbandoned`. **Proposed microcopy:**
   - mode 1 — **en:** "Lock the full amount now — fastest acceptance." · **ar:** «اقفل المبلغ كاملاً الآن — أسرع قبول.»
   - mode 2 — **en:** "Lock a 10% good-faith deposit — shows you're serious." · **ar:** «اقفل عربون جدّية 10٪ — يُظهر جدّيتك.»
   - mode 3 — **en:** "Lock nothing until your developer accepts." · **ar:** «لا تقفل شيئاً حتى يقبل مطوّرك.»
   - cancel — **en:** "Cancel anytime before your developer accepts — your USDT returns in full." · **ar:** «ألغِ في أي وقت قبل قبول المطوّر — تعود أموالك كاملةً.»
   (No custody/guarantee/refund/escrow/«مضمون/ة»; the smart contract is the actor — §18.)
4. **Docs + gates:** update the docs listed above; `forge build/test/coverage`, slither clean, en+ar
   screenshots (§16), §17 fee + §18 legal CI green; independent audit before any mainnet (P5).

## Tunable sub-decisions (recommended defaults; owner may adjust in one line)

- **T1 — seriousness enforcement for mode 2 flake:** **reputation counter (recommended)** vs **forfeit the
  10% to the developer** (stronger "teeth", but introduces penalty/custody semantics + legal surface, and
  conflicts with §18 "no money without delivered work"). *Default: reputation counter.*
- **T2 — funding window length (modes 2/3 post-approval):** *Default: 48h.*
- **T3 — defaults & instant-order:** Gig→Full, Open-Job→Deposit-10%; Gig instant-order (skip dev-confirm) =
  *off by default* (keeps a cancel-before-approval window even on Gigs).

On confirmation of T1–T3 (or acceptance of the defaults), implementation proceeds under TDD with the attack
scenarios in §Implementation/2.
