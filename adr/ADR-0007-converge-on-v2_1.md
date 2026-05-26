# ADR-0007 — Converge on DevSwapEscrowV2_1 as the Single Contract (one contract, two doors)

- **Status:** Accepted (owner-authorized 2026-05-23 — "لك الصلاحية الكاملة في اختيار أفضل الممارسات")
- **Date:** 2026-05-23
- **Relates:** DOC-1 §6 (unified model), DOC-4 §1.4, **DOC-5 §2 (this ADR is DOC-5's H0)**; ADR-0002
  (hourly out of scope); ADR-0003 (arbiter hardening). Supersedes the V1-vs-V2 split for the UI layer.

## Context

The hybrid marketplace is **half-built but split into two parallel UIs**:

- **V1 `DevSwapEscrow`** — single task = single payment — is **live on testnet** and powers the main UI
  (`/explore`, `/post`, `/task/[id]`). It has **no on-chain reputation**.
- **V2.1 `DevSwapEscrowV2_1`** — milestone escrow + **on-chain reputation** (`getReputation`) + **tiered
  disputes** (ADR-0003) — is **deployed on testnet** but gated behind `NEXT_PUBLIC_V2_ENABLED`, duplicating
  explore/post/detail/admin under `/v2/*`.

The competitive analysis already ratified the model that resolves this: **"Gig" and "Job" are the same
on-chain object** — a `Job` with milestones (DOC-1 §6). V2.1's `createJob(uint256[] milestoneAmounts,
string metadataHash)` **is** that object: one milestone = a Gig; many = an open Job. Maintaining two UIs
(and two contract libs, `contracts.ts` + `contractsV2.ts`) is integration debt that grows the longer it
persists, and it strands reputation + tiered disputes behind a flag.

**Timing makes this cheap now:** mainnet is **owner-deferred** (STATE.md) and V1 lives only on testnet, so
there are **no real funds to migrate** — the cost of converging rises every week two UIs coexist.

## Decision

1. **`DevSwapEscrowV2_1` is the single contract** behind the entire app. Both hybrid doors map to it:
   - **Gig door** (Fiverr-style, fixed scope) → `createJob([oneAmount])`.
   - **Open-Job door** (Upwork-style, proposals) → `createJob([m1, m2, …])` after the client accepts a
     **free off-chain proposal** (SYNTH-4 §9; no Connects-style pay-to-bid — DOC-1 §7).
2. **Unify the UI surfaces** (per DOC-5 H1–H7): one data adapter (`web/lib/jobs.ts`), one Explore, one
   builder (`/create`), one detail hub (`/job/[id]`), one role-aware dashboard.
3. **Retire V1's separate UI** in a **phased, non-destructive** way: legacy V1 testnet tasks become
   **read-only** (a read adapter maps a V1 task → a 1-milestone Job view); **no funds are migrated**.
   Removal of `/v2/*` and the `NEXT_PUBLIC_V2_ENABLED` flag happens **only after** the unified surfaces
   land and are verified.
4. **Carry forward ADR-0002:** convergence does **not** add hourly — V2.1 stays milestone-only.

**Out of scope of this ADR:** mainnet deployment of V2.1 (owner-deferred behind the P5 audit/multisig
gate) and any change to the deployed V1 testnet contract (left as-is, read-only).

## Consequences

- **Unlocks shipped-but-hidden value:** on-chain reputation (DOC-3 Wedge 3) and tiered disputes (Wedge 4)
  appear in the main experience instead of behind a flag.
- **Less surface to audit & maintain:** one contract lib, one set of screens; eliminates V1/V2 drift.
- **Migration risk: low** (testnet only, no fund movement); a unit test covers the V1→1-milestone read map.
- **Sequencing:** additive phases (DOC-5 H1–H6) proceed immediately; the **destructive** step (deleting
  `/v2/*` + flag) is the last step and is explicitly gated on the unified UI being verified green.
- **Reversibility:** until the destructive step, V1 and V2.1 still both function; the convergence is
  additive-first by design.
- **Follow-ups:** flag removal tracked in DOC-5 H8; `BLOCKED.md` notes any key-gated piece (e.g. proposals
  needing Supabase).
