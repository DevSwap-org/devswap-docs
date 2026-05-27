# ADR-0003 — V2.1 dispute hardening: arbiter registry, timelock, and snapshot

- **Status:** Accepted (one sub-decision flagged for owner ratification — §4)
- **Date:** 2026-05-22
- **Deciders:** Maintainer team
- **Relates to:** ADR-0002 (V2), `contracts/src/DevSwapEscrowV2_1.sol`, `SECURITY-AUDIT.md` §6
- **Supersedes (operationally):** the dispute model of the original `DevSwapEscrowV2`

## Context

A pre-deployment review of `DevSwapEscrowV2` found three weaknesses in tier-2 dispute resolution,
all centered on a single trusted key:

1. `resolveDispute` was `onlyOwner` (and the simple arbiter registry made the owner an **implicit
   arbiter**) — the owner could unilaterally decide any dispute.
2. `setArbiter` took effect **immediately** — the owner could add an arbiter in the same block.
3. Disputes carried **no snapshot** of who was eligible to judge them — `resolveDispute` checked
   membership at *resolution* time.

Together these let a compromised or malicious owner key insert a "puppet" arbiter and swing a live
dispute, or self-resolve, with no delay and no audit window.

## Decision

Build `DevSwapEscrowV2_1` (new contract; the original V2 is deprecated and removed from the UI) with
a hardened arbiter system.

### 1. Separate the owner from the arbiter role (threat model: EOA compromise)

`_checkArbiter` requires `isArbiter[caller]` only — the owner is **not** an implicit arbiter. The
owner manages the registry (add/remove) but cannot resolve a dispute unless explicitly registered.

**Why:** the deployer/owner is a hot **EOA** today (a single private key; multisig is a P5 gate). If
that key is compromised, an owner-as-judge model means instant theft of every disputed milestone.
Separating the roles means a compromised owner still cannot *directly* resolve disputes — it can only
*queue* an arbiter, which is delayed and observable (§2, §3). Defense in depth before the multisig
lands.

### 2. Timelock arbiter additions at 48h (vs 24h / 7d)

Adding an arbiter is `queueArbiter` → wait `ARBITER_TIMELOCK = 48 hours` → `executeArbiter`, with
`cancelArbiterChange` to abort a pending add. **Removal is immediate** (`removeArbiter`).

**Why 48h specifically:**
- **24h** is too short — it can be slept through over a single night/weekend; counterparties and
  monitors may not notice or react in time.
- **7d** is operationally too slow — onboarding a legitimate new arbiter (e.g. scaling the panel)
  shouldn't take a week, and a long queue tempts keeping a large standing arbiter set (worse).
- **48h** is the standard governance-timelock floor (matches common DeFi timelocks) and spans a
  full business cycle: any queued arbiter is visible on-chain for two days, long enough for the
  counterparty to `raiseDispute` / exit or for the owner to `cancelArbiterChange` if the key event
  was malicious. It is also the lower bound we will keep when the owner becomes a multisig.

**Why removal is immediate (asymmetric):** the timelock exists to stop *adding power*; revoking a
compromised or misbehaving arbiter is a safety action that must be instant. Slow removal would be a
liability, not a protection.

### 3. Snapshot dispute-open time; gate eligibility on it

`raiseDispute` records `disputeRaisedAt` (uint64). `resolveDispute` requires
`isArbiter[caller] && arbiterSince[caller] <= disputeRaisedAt`. An arbiter whose `arbiterSince` is
after the dispute opened is rejected with `ArbiterNotEligible`.

**Why:** this is what actually closes the "puppet arbiter mid-dispute" hole. Even if an attacker
controlled the owner key, any arbiter they queue is (a) delayed 48h and (b) — because
`arbiterSince` is set at `executeArbiter` time — **never eligible** for a dispute that was already
open. Combined, an attacker cannot manufacture a judge for an in-flight dispute at all.

### 4. Removed-arbiter eligibility — RATIFIED: option (A)

**Status: ratified by the owner (2026-05-22). The current V2.1 implementation is correct as-is; no
contract change.**

The check `isArbiter[caller] && arbiterSince[caller] <= disputeRaisedAt` means a **removed** arbiter
(`isArbiter = false`) **cannot** resolve any dispute, including ones that predate their removal.

- **(A) — "current membership required" (CHOSEN, implemented):** removal fully revokes judging power.
- **(B) — "eligibility survives removal" (REJECTED):** a removed arbiter could still resolve disputes
  open at the time of removal.

**Why (A) — four reasons:**
1. **Conservative default.** Removal should mean *full* revocation. The reason you remove an arbiter
   (key compromise, misbehavior, collusion) is exactly the reason you must NOT let them keep
   finishing off live disputes — (B) would let a freshly-revoked, possibly-malicious arbiter still
   swing every dispute that was already open.
2. **Clean kill-switch semantics.** `removeArbiter` is the emergency revoke. It must nullify *all*
   power in one immediate, easy-to-reason-about action — not leave a tail of "but they can still
   resolve these N open disputes."
3. **Simpler, smaller attack surface.** (A) needs no `arbiterRemovedAt` bookkeeping and no extra
   branch in the hot `resolveDispute` path. Fewer state variables and conditions = less to audit and
   fewer ways to get the eligibility math wrong.
4. **Operationally mitigated.** With ≥3 arbiters (policy below), removing one never strands an open
   dispute — a peer eligible at `disputeRaisedAt` resolves it. The theoretical downside of (A) is
   eliminated by operations, so the security win is free.

**Edge case (intentional):** if *every* eligible arbiter for an open dispute is removed, that dispute
**freezes** (no one can resolve it) until the owner queues + executes (48h) a new arbiter — who, by
the snapshot rule, is then **not** eligible for that already-open dispute. In other words, a fully
de-arbitered open dispute can become permanently unresolvable on-chain. This is **by design**: it is
the safe failure mode (funds stay escrowed; nothing is mis-paid) and the price of the strict
kill-switch. It is prevented operationally, never reached in practice — see the policy.

**Operational policy (enforced in `docs/RUNBOOK.md`):** **maintain ≥3 active arbiters at all times.**
Removing one then always leaves ≥2 eligible for any open dispute, so the freeze edge case is never
hit. Arbiter count is a weekly-monitored operational metric.

**Why (B) is rejected:** it inverts the kill-switch (a revoked arbiter retains power over the exact
disputes you most want to take from them), adds `arbiterRemovedAt` state + an extra eligibility
branch, and widens the audit surface — all to "solve" a problem the ≥3-arbiter policy already solves.

**Possible future (V2.2, OUT OF SCOPE now):** an `emergencyArbiter` role with a shorter **24h**
timelock for fast incident response (e.g. the primary arbiter is unreachable during an active
dispute), kept distinct from the standard 48h add path. Not built now — it adds a second privileged
path and must be designed against the same snapshot rule; revisit only if operations show a real need.

### 5. Clean bootstrap via constructor (no one-shot hack)

The constructor takes `initialArbiter` and registers it immediately (`isArbiter = true`,
`arbiterSince = block.timestamp`) — no timelock for this single bootstrap entry.

**Why:** without it there is a chicken-and-egg problem — no one could resolve the first dispute until
48h after the first `queueArbiter`. The alternative (a one-time "first add is instant" flag in
`queueArbiter`) is stateful, easy to get wrong, and a classic audit foot-gun. A constructor parameter
is explicit, immutable in intent, and trivially auditable.

## Consequences

**Positive**
- A compromised owner EOA can no longer self-judge or insert a puppet arbiter into a live dispute.
- All arbiter *additions* are observable on-chain for 48h; *removals* are instant.
- 100% test coverage incl. all five mandated cases; Slither 0 high/medium; solvency invariant intact.

**Negative / trade-offs**
- Operational care: keep ≥2 eligible arbiters so removing one never strands an open dispute (see §4).
- The original V2 is **deprecated**: it is not referenced by the UI (the `/v2` routes and
  `ESCROW_V2` point at the V2.1 deployment) and its PRs (#4/#5) are superseded. The deployed original
  V2 testnet instance is abandoned (testnet, valueless).

**Out of scope:** tier-3 decentralized jury (future) — **broad analysis + the A0→A6 arbitration best-practice
roadmap now live in [`docs/DISPUTE-RESOLUTION.md`](../DISPUTE-RESOLUTION.md)** (this ADR is its A0 baseline);
the P5 mainnet gate (audit + owner→multisig +
timelock + LP lock) is unchanged and applies to V2.1 independently.
