# DevSwap — Trust, Incentives & Decentralization (strategy / design)

> **What this is.** A strategy / design document (peer to [`DISPUTE-RESOLUTION.md`](DISPUTE-RESOLUTION.md), [`TOKENOMICS.md`](TOKENOMICS.md)) answering seven design questions that span four areas: **(A) arbitration economics** (juror availability, fallback, how the arbiter is paid, penalties for a wrong party), **(B) on-platform communication, file exchange & anti-disintermediation**, **(C) `$DSWP` demand-side utility & in-platform marketing** (with a permanent free tier), and **(D) decentralised developer verification**.
>
> **Status: analysis + plan only — no contract / web code changed.** Every mechanism here is future work, TDD + audit + governance-ratified (mainnet behind the gates in [`SECURITY-AUDIT.md §1`](SECURITY-AUDIT.md)); each becomes its own ADR when ratified. Reserved: **ADR-0016** (token demand utility), **ADR-0017** (messaging + anti-disintermediation), **ADR-0018** (decentralised verification). Arbitration phases keep their reserved **ADR-0010 .. 0015** ([`DISPUTE-RESOLUTION.md §6`](DISPUTE-RESOLUTION.md)).
> All 7 design forks were ratified in May 2026. The values they fixed are documented in the relevant ADRs.
> **Governance layer.** The long-term goal is to move all parameters in this document to DAO on-chain governance. See [`GOVERNANCE.md`](GOVERNANCE.md) for the G0 → G4 decentralisation roadmap + 3-tier timelock architecture. At G3+, Tier-1 / Tier-2 votes replace ADR ratification for every parameter in Areas A – D.
>
> **Constraints (do not drift):** **non-custodial** (the smart contract + parties + independent attesters / arbiters bear risk, not "we" — see [`ADR-0006`](adr/ADR-0006-non-custodial-positioning.md)); **validate-first** (`$DSWP` deferred until GMV is proven — [`TOKENOMICS.md`](TOKENOMICS.md); all of Area C is therefore a *post-token-launch* phase); **securities-safe** (no staking-for-yield / farming / vaults / price-appreciation promise — [`TOKENOMICS.md`](TOKENOMICS.md)); fee **3 % (1.5 % platform + 1.5 % buyback), dev 97 %**; UI copy avoids custody / guarantee / refund language; en / ar parity. **This is mechanism design, NOT legal advice** — independent digital-asset counsel before mainnet is deferred-but-mandatory.
> Recommendations are tagged **design inference / engineering judgment**; external protocols (Kleros, UMA, EAS, Human Passport, etc.) are referenced as mechanism categories for orientation, each requiring its own due-diligence + audit before adoption.

---

## TL;DR

DevSwap is non-custodial, so the answer to every question is **incentives + on-chain transparency +
progressive decentralization**, never a central authority. (A) Arbiters: availability via ≥3-policy →
staked pool → never-frozen fallback; paid by a **loser-funded dispute fee**, *not* from the dev's clean
97%; the wrong party forfeits the disputed milestone + takes a public reputation counter (no slashing of
out-of-escrow funds — impossible). (B) You **cannot fully stop** off-platform leakage without custody —
make on-platform **strictly dominant** via carrots (escrow + dispute protection + portable reputation exist
only on-platform), treat contact-masking/bans as low-investment theater. (C) Post-launch token demand via
**consumptive sinks** (dev pays to boost visibility, client pays to prioritize) that are a **capped,
labeled multiplier on merit**, with a **permanent full-feature free tier**; spent `$DSWP` auto-burns,
**never marketed as a return**. (D) Layered decentralized verification (wallet↔GitHub → humanity gate →
**on-chain work history as the load-bearing skill credential** → optional **stake-backed, slash-on-fraud**
peer attestation); **reject** "subject pays verifier + platform takes a cut."

---

# Area A — Arbitration economics (Q1, Q3 specifics)

> Builds on [`DISPUTE-RESOLUTION.md`](DISPUTE-RESOLUTION.md) (the A0→A6 roadmap) and
> [ADR-0003](decisions/ADR-0003-arbiter-hardening.md) (the shipped A0 baseline). This section answers the
> *economic* sub-questions that roadmap defers to phases A3–A6.

## A.1 Guaranteeing arbiter availability — three layers, decentralizing over time

| Layer | Mechanism | Phase |
|---|---|---|
| **Now (A0)** | RUNBOOK **≥3 active arbiters** policy (weekly-monitored); removing one always leaves ≥2 eligible for any open dispute (ADR-0003 §4). Availability is *operational*. | ✅ shipped |
| **Mid (A4/A6)** | **Incentivize a pool** so availability is *economic*, not operational: arbiters earn a dispute fee (A.3) and/or stake to be eligible; a larger eligible set ⇒ random assignment can always seat a panel. A reputation-gated + staked juror pool (Kleros-style **mechanism category**) makes "enough judges exist" a market outcome. | ADR-0013 (A4) / ADR-0015 (A6) |
| **Edge (A5)** | **No-permanent-freeze fallback** (closes gap **G7**): if a dispute has **no eligible arbiter** past a long timeout (e.g. 30–60d), a **governed** last-resort path resolves it — a multisig-appointed emergency arbiter via a special **audited, snapshot-safe** path, *or* a conservative deterministic default (**return-to-client**). Funds are **never lost and never frozen forever**. | ADR-0014 (A5) |

**Why this is the honest answer:** a decentralized protocol can't *command* humans to show up. It can only
(1) make showing up profitable, (2) make the eligible set large enough that random assignment rarely fails,
and (3) guarantee a safe terminal state when it does fail. The current safe failure mode is "funds stay
escrowed, nothing mis-paid" (ADR-0003 §4) — A5 upgrades that to "always resolvable."

## A.2 The fallback when no arbiter exists

Per A.1 Edge / **A5 / G7**. Two governed options (owner fork at ADR-0014):
- **(a) Multisig emergency arbiter** — a special path, behind the same `disputeRaisedAt` snapshot rule
  (so it can't be abused to back-date a puppet judge), usable only after the long timeout. Preserves human
  nuance for the stuck case.
- **(b) Conservative deterministic default** — after the timeout, the contract returns the disputed
  milestone **to the client** (the most conservative: it never *pays out* on un-adjudicated work). Zero
  human, zero discretion, but blunt.

*Design inference:* ship **(b) as the floor** (always-available, trustless) and offer **(a) as an optional
faster human path** while the panel decentralizes. Both are audit-gated.

## A.3 How the arbiter is rewarded — **NOT from the dev's 97%** (Q1)

This is the key clarification. **The 97/1.5/1.5 split is the *happy-path settlement* split** (§17) and is
**untouched** when there's no dispute. Arbiter pay is a **dispute-only** economic layer:

- **Source = a loser-funded dispute deposit** (closes **G2**, anti-griefing). Raising a dispute posts a
  small **refundable** deposit (in USDT or `$DSWP`). The **winner's deposit is returned**; the **loser's**
  funds the arbitration (arbiter fee + protocol slice). So honest parties pay nothing; only the wrong party
  pays — and the arbiter is paid by **the party who caused the dispute**, not by the developer's clean
  earnings.
- **Optional second source = a small slice of the *disputed* amount** (e.g. an `ARBITRATION_FEE_BPS` taken
  *only* from milestones that actually go to ruling). If used, it must be a **separate, dispute-only
  constant** that **does not** alter the headline "97% to the developer" for normal settlements — and the
  §17 consistency cascade must document it as a *dispute fee*, not the settlement fee. **Owner fork.**

**Why not from the 97 %:** taxing the developer's normal earnings to fund a dispute machine they may never use is both unfair and undermines the "97 % to the dev" headline. The polluter pays. *(Maps to [`DISPUTE-RESOLUTION.md §4.2 + §4.5`](DISPUTE-RESOLUTION.md); lands in A3 / A4.)*

## A.4 Penalty when the **developer** is proven wrong (Q1)

Non-custodial reality: **we can only act on funds already in escrow** — there is no way to "fine" a dev's
external wallet. So the penalty is:
1. **Forfeit the disputed milestone** — funds return to the client per the ruling. With **partial/split
   rulings** (closes **G3**, A3) a *partially*-delivered job can split (e.g. 60/40) rather than all-or-nothing.
2. **Public `disputesLost` counter increments** ([ADR-0003] [:69]) — a permanent, portable reputation hit
   (the on-chain-reputation MOAT working as a deterrent).
3. **If the dev was the one who wrongly escalated** (e.g. lost an appeal): forfeit their **dispute/appeal
   deposit** (A.3 / A6).
4. *(Future, A4)* if the dev posted a **bid/commitment stake** (Braintrust-style **mechanism category**),
   that stake is **slashable** on a proven-bad outcome — the only way to reach "skin in the game" beyond the
   escrowed milestone, and only on funds the dev *chose* to stake.

## A.5 Precautionary plan when the **client** is proven wrong

Protecting the dev from **extortion-by-dispute** is symmetric and is the main reason for the deposit:
1. **Release the disputed milestone to the developer** (the ruling pays the honest dev).
2. **Client forfeits their dispute deposit** (A.3 / G2) — raising a frivolous dispute has a real cost.
3. **Client-side public counter** — e.g. `disputesLostAsClient` / the existing `commitmentsAbandoned`
   ([ADR-0009](decisions/ADR-0009-funding-trigger.md)) — clients build a reputation too; serial bad-faith
   clients become visible to devs deciding whether to take their jobs.
4. **Pre-dispute friction** that prevents the situation: **dispute cooling-off** (UX-9) + a **first-class
   revision request** (closes **G8**, A1) + **Tier-1 mutual settlement** (closes **G4**, A2) — so an
   "almost-right" delivery is fixed or split *before* anyone needs an arbiter.

> **Net:** honest parties are unaffected; the wrong party (dev *or* client) pays the dispute cost + takes a
> reputation hit. This symmetry is the anti-griefing design — neither side can weaponize the dispute button.

---

# Area B — Communication, file exchange & anti-disintermediation (Q2, Q3 comms)

> **Honest framing first.** In a non-custodial model the protocol has **surrendered the strongest retention lever** (withholding funds). Web2 contact-masking and ToS off-platform bans are, per the published research, largely theatre — bypassed by users and damaging to trust, and near-unenforceable against pseudonymous wallets. Pretending we can *prevent* disintermediation would be dishonest. The realistic goal is to make the **on-platform path strictly dominant** and accept some leakage on one-off small deals while winning repeat + high-value work. *(Public sources: Sharetribe marketplace-leakage guide; INFORMS 2025 marketplace dynamics survey. Web3 precedent: Braintrust, TalentLayer / Kleros.)*

## B.1 How the two parties talk **on-platform** (Q2)

In-app messaging is on the roadmap as **ADR-0017** (planned). Design intent when built:
- **Threaded, milestone-linked, searchable** messaging tied to the job ID — it must genuinely *beat* WhatsApp / email (in-context links to the milestone, deliverable, and the funding tx), not merely restrict it. This is the carrot: the conversation lives next to the money and the evidence.
- **Storage:** off-chain (Supabase, the existing mirror) with **optional E2E encryption**; only a **message-thread hash** need ever touch chain (as dispute evidence). Keeps it cheap and private.
- **Optional in-app video** for scoping calls — later.
- **Identity-reveal timing:** reveal contact identities **at contract start, not before** — so the escrow carrot is locked in *before* contact details change hands.

## B.2 File / deliverable exchange (Q2)

- **IPFS** for briefs + deliverables (already the project's evidence substrate); files encrypted, with the
  **content hash anchored on-chain**. *(design inference)*
- **The hash is itself an anti-disintermediation carrot:** an on-platform delivery has a **tamper-proof,
  timestamped evidence trail** an arbiter can rule on; an off-platform delivery (WeTransfer/email) has
  **none**. Going around the platform means going *without* provable delivery — a real, non-coercive cost.
- Full files are revealed to an **arbiter only on dispute**; otherwise only the parties hold the decryption.

## B.3 Anti-disintermediation levers, ranked (Q2) — carrots win, sticks are theater

| # | Lever | Type | Strength |
|---|---|---|---|
| 1 | **Escrow + dispute protection only for on-chain deals** — go off-platform, lose all smart-contract protection | carrot | **strong** (the single most defensible non-custodial lever; mirrors TalentLayer) |
| 2 | **Portable on-chain reputation accrues only from on-platform settled jobs** — leaving forfeits the asset | network-effect | **strong** ("highly effective" — Sharetribe; core to Braintrust/TalentLayer) |
| 3 | **On-chain deliverable-hash evidence trail** (B.2) | carrot | **strong** (design inference) |
| 4 | **`$DSWP` visibility/marketing utility exists only on-platform** (Area C) | network-effect | **strong** |
| 5 | **Platform-only value-added services** (CI checks, invoicing, analytics, badges) | carrot | strong (Sharetribe) |
| 6 | **Token-reward participation loop** (buyback-burn already aligns good on-platform citizens) | network-effect | strong (Braintrust model) |
| 7 | **Structured in-app messaging + files + in-context links** (B.1) | carrot | strong→retention |
| 8 | **Contact-info masking pre-deal** (auto-redact obvious emails/phones in chat) | stick | **weak/theater** — bypassed; keep lightweight, do not over-invest |
| 9 | **ToS off-platform penalties / bans** | stick | **weak/theater** — near-unenforceable for pseudonymous wallets; norm-signaling only |

> **Strategy:** lead with carrots (#1–3) + network effects (#4–6); treat sticks (#8–9) as low-cost
> norm-signaling. **The moat is reputation + protection-only-on-platform, not coercion.**

---

# Area C — `$DSWP` demand-side utility & in-platform marketing (Q5, Q6)

> **Sequencing:** this entire area is **post-token-launch** — `$DSWP` is deferred until GMV is proven
> ([`TOKENOMICS.md §9`](TOKENOMICS.md)). It does **not** change the MVP (USDT-fee-only). **This is mechanism
> design, not legal advice** — counsel before mainnet is mandatory.

## C.1 Making devs & clients *need* the token — consumptive sinks (Q5)

Today `$DSWP`'s only role is buyback-burn (a passive value sink). To create **demand**, add **consumptive
use** — spend-and-it's-gone, like ad credits:
- **Developer visibility boost:** a dev spends `$DSWP` to appear in **promoted** slots / a "top developers"
  rail — *in-platform marketing*.
- **Client priority boost:** a client spends `$DSWP` to lift their job into a **priority** list (more dev
  eyeballs, faster proposals).

**Why consumptive sinks are the securities-safer choice:** the buyer's intent is **consumption** (a service
purchase), not profit from others' efforts. This is materially lower-risk than staking/yield — but **utility
does not erase Howey** (SEC LBRY/Uniswap posture; Skadden 2025). The framing that keeps it safe: **a
consumed visibility *service fee*, priced in `$DSWP`** — *never* an investment, *never* tied to price.

## C.2 Pay-to-rank **without** wrecking trust (Q5/Q6) — the critical guardrails

Every major ad platform makes paid boost a **multiplier on merit, never an override** (Google Ad Rank = bid
× **Quality Score**; Amazon ranks on relevance + convert-likelihood, not bid alone). DevSwap must do the same:

1. **Rank = on-chain reputation/merit × boost multiplier**, with the **multiplier capped** (e.g. ≤1.5–2×). A
   poorly-rated dev **cannot** buy the top no matter how much `$DSWP` they burn.
2. **Label every boosted slot "Promoted"** — clear and conspicuous (FTC endorsement guides; legal-safe
   copy).
3. **Cap promoted slots per page** (e.g. ≤2 of top 8 — mirrors Upwork's 4-slot cap).
4. **Reputation floor:** no boosting brand-new/unrated accounts to the top; boost **decays over time**.
5. **Auction vs fixed price:** start with **transparent fixed-price boost packages** while liquidity is thin
   (predictable, newcomer-friendly, no bidding-war spiral); migrate to a **capped second-price auction** only
   after GMV depth. Always publish the pricing formula. **Owner fork** (start mode + migration timing).

## C.3 Where spent `$DSWP` goes — the securities-critical fork (Q6)

The sharp risk: **burning the spent token re-introduces a profit angle** (deflation benefiting holders →
"common enterprise / expectation of profit"). Buyback-burn survives Howey **only** when no payout/yield is
promised and benefit is *indirect* scarcity from the protocol's own rules-based actions (Skadden; Tiger
Research 2025). Therefore:
- **Recommended (safest):** spent boost-tokens are **consumed/burned on spend via the *same automatic,
  rules-based path*** as the 1.5% deal fee — **no discretionary timing, no holder distribution.** Deflation
  stays a silent byproduct of consumption.
- **Avoid:** routing spent tokens to a **discretionary treasury** (looks managerial) **or** redistributing to
  holders (looks like a **dividend** — the strongest Howey signal).
- **Marketing rule (hard):** **never** tie boosts/burn to price appreciation in any copy — that single
  framing error is the biggest controllable risk. Keep "burn" in technical docs only (consistent with §18).
- **Owner/legal fork:** burn-spent-boosts vs immediate-consume-only (cleanest: the token is *consumed*, full
  stop; deflation is never marketed).

## C.4 The permanent free tier + how the token reflects to each party (Q6)

**Free tier (non-negotiable):** the permanent free tier grants the **full ability to list, bid, transact,
build reputation, and rank purely on merit.** `$DSWP` only ever buys **additional visibility / convenience**
— it **never unlocks the ability to participate.** This is the structural defense against "pay-to-play": an
organic merit path always exists beside the promoted slots.

| Party | Token utility (consumptive, optional) | Free tier always gets |
|---|---|---|
| **Developer** | boost visibility, "top developers" rail, profile badges, more concurrent promoted slots, analytics | list, bid, transact, full on-chain reputation, merit ranking |
| **Client** | priority placement for a job, featured brief, faster proposal funnel | post jobs, receive proposals, fund escrow, dispute protection |
| **Arbiter / juror** | *(A6)* **stake `$DSWP` to be eligible** for the juror pool + earn dispute fees (A.3) — note: this is **stake-to-work**, more defensible than stake-for-yield, but still a **legal fork** to vet | eligibility via the appointed-panel path (no stake needed at A0) |

**The flywheel (honest version):** two-sided consumptive demand (dev boost + client priority) → tokens
**needed to use real features** → automatic rules-based buyback-burn on the same rails → recurring,
*activity-tied* buy-pressure. Demand comes from **needing the token to use the feature**, not from a yield
promise. (Cautionary precedent: Axie/SLP collapsed when its sink failed — sinks must be tied to genuine
core-product usage, not artificial.)

---

# Area D — Decentralized developer verification (Q7)

> Core principle: **the platform must not be the central authority that certifies "real developer"** (both a
> liability and a centralization risk). Verification = a portable **claim/attestation**, never a platform
> guarantee. Seed already exists: **GitHub OAuth + wallet↔GitHub fresh-signature binding** (IDX-2).

## D.1 Recommended layered architecture (Q7)

1. **Identity binding (have today):** wallet↔GitHub via fresh signature → expose as an **EAS (Ethereum
   Attestation Service)** attestation. EAS is neutral infra: value comes from *who attests*, not the protocol
   — exactly DevSwap's liability posture.
2. **Humanity / sybil gate:** read a **Human Passport** (ex-Gitcoin Passport) "Unique Humanity Score"
   (aggregates GitHub/BrightID/PoH stamps); optional **World ID** stamp for high-assurance. DevSwap **reads**
   the score; it never issues it. No hardware mandate, no platform liability.
3. **Skill signal — the load-bearing layer (already minted):** the **on-chain work history** DevSwap already
   produces (milestones paid, disputes lost, total earned, keyed to the verified wallet) **is** the strongest,
   cheapest, sybil-costly credential — faking it requires real USDT settlements. Expose it + GitHub activity
   as EAS attestations.
4. **Optional staked peer-attestation** (see D.2) for an extra, higher-assurance skill claim.
5. **Client-side composition:** the client sees the **bundle of independent attestations** and decides. The
   platform is a **neutral registry**, not the verifier.

## D.2 Verdict on the owner's "dev pays a verifier, split with the platform" idea (Q7)

**Modify — mostly reject the payment direction.** Two structural flaws:
- **Payer = subject ⇒ rubber-stamping.** A verifier paid by the person being verified is structurally biased
  toward approval (well-documented; *disclosing* the conflict can even *worsen* it — "moral licensing").
- **Platform taking a cut of an approval fee makes the platform a financially-interested party in
  verification decisions** — the exact centralization/liability the project is built to avoid.

**Adopt instead — stake-backed attestation (skin in the game):**
- A verifier **stakes** (`$DSWP`/USDT) to attest a dev's skill, and earns a **small bounty only if the dev
  performs**.
- **Slashing on later-proven fraud:** if the attested dev is shown fraudulent (lost dispute / fraud proof),
  the attester's stake is **slashed** and redistributed to the harmed client/challenger. The incentive flips
  from "approve for a fee" to "approve only what you'd bond on."
- **"Who verifies the verifiers"** is solved with **Kleros-style mechanics** (mechanism category): random
  stake-weighted selection + commit-reveal voting + Schelling-coherence rewards. Attesters accrue **their own
  on-chain reputation** (attestation accuracy), making them sybil-costly.
- **Platform revenue stays on settlements only — never on approvals.** This keeps the platform out of the
  verification decision entirely.

## D.3 Liability framing (Q4 + Q7)

- Copy says **"attested by [attester]"**, **never** "DevSwap-verified" (the
  actor is the attester, not the platform).
- DevSwap publishes **schemas + the registry (the protocol)**; humans/attesters make **claims**; clients
  judge. Verification is a portable attestation, **not** a platform guarantee.

## D.4 Failure modes → mitigations

| Failure | Mitigation |
|---|---|
| Verifier rubber-stamps for a fee (payer-bias) | **subject never pays the verifier**; stake + slash-on-fraud |
| Sybil verifiers | stake requirement + verifier on-chain reputation + humanity gate on attesters |
| Collusion / bribery | random stake-weighted selection + commit-reveal (Schelling) |
| Wealth concentration | cap per-attester weight; reputation-weight, not pure stake |
| Griefing challenges | challenger must bond; loser pays |
| "Who verifies the verifiers" regress | recursive Kleros-style coherence rewards; no central arbiter |
| Platform pulled into liability | fee only on settlements; "attested by X" copy; neutral registry |

---

# Area E — Platform liability & decentralisation posture (Q4)

> Foundation: [`ADR-0006`](adr/ADR-0006-non-custodial-positioning.md) (non-custodial positioning) + the project's legal-safe language policy. This is positioning + progressive decentralisation, not a substitute for legal review; qualified counsel before mainnet is mandatory.

How DevSwap limits its responsibility *honestly* (each maps to an area above):
1. **Funds:** the **smart contract** holds/releases USDT — not the company ("we don't hold or process
   payments", ADR-0006).
2. **Disputes:** **independent arbiters/jurors** rule on-chain — not platform support. The platform doesn't
   decide; it can't (the owner is *not* an implicit arbiter — ADR-0003 §1).
3. **Arbiter availability:** guaranteed by **economic incentives + a never-frozen fallback** (Area A),
   **not** a platform promise.
4. **Arbiter collusion:** mitigated by **random assignment + stake/slash + appeal + public track-record**
   (Area A / DISPUTE-RESOLUTION §4) — not a platform guarantee of integrity.
5. **Verification:** **attesters** make claims; the platform is a **neutral registry** (Area D).
6. **Transparency:** every ruling/settlement is a **public on-chain tx** — auditable by anyone.
7. **Progressive decentralization (the real shield over time):** appointment + token + registry governance
   move **owner-EOA → multisig (3-of-5) → DAO** (P5 gate + DISPUTE-RESOLUTION best-practice #11). The trust
   surface shrinks on a schedule, and the docs are **honest that full trustlessness is a journey** —
   arbitration remains the main residual centralization until A6.

---

## Phased plan (what to build, in what order)

> All future work; TDD + slither + en/ar + §17/§18 + securities-safe; **independent audit + owner→multisig
> before mainnet (P5)**. Token-dependent items (Area C, parts of A/D) come **after** the validate-first token
> launch.

| Phase | Scope | Depends on | ADR |
|---|---|---|---|
| **T1 — Prevention + settlement** | revision loop (G8), dispute cooling-off, **Tier-1 mutual settlement** (G4) | — | ADR-0010/0011 |
| **T2 — Fair, anti-griefing Tier-2** | **dispute deposit** (loser-funded, G2) → **funds the arbiter** (A.3); **partial/split rulings** (G3); evidence window; random assignment + conflict rules (G9) | T1 | ADR-0012 |
| **T3 — Messaging + files** | threaded milestone-linked chat; IPFS encrypted files + on-chain deliverable hash; identity-reveal-at-contract | backend | **ADR-0017** |
| **T4 — Arbiter incentives + freeze fallback** | arbitration fee / stake+slashing + public track-record (G5, A.3); **no-permanent-freeze** governed fallback (G7, A.1/A.2) | T2 | ADR-0013/0014 |
| **T5 — Verification layer** | EAS attestations (wallet↔GitHub + work-history); Human Passport gate; optional **stake-backed peer attestation** (D.2) | — (stake parts need token) | **ADR-0018** |
| **T6 — Token demand utility** *(post-launch)* | dev visibility boost + client priority boost; **capped, labeled, merit-multiplier**; **permanent free tier**; auto-burn-on-spend; fixed→auction migration | `$DSWP` launch (validate-first) | **ADR-0016** |
| **T7 — Decentralized escalation + governance** | bounded appeal → Tier-3 jury (value-gated); registry/token governance owner→multisig→DAO | T4 | ADR-0015 (A6) |

---

## Design forks (need ratification before the relevant phase)

1. **Arbitration fee model (A.3):** loser-funded deposit only, *or* deposit + a small dispute-only `ARBITRATION_FEE_BPS` slice of the disputed milestone? (If the latter, document as a **dispute fee**, not the settlement fee, in the consistency cascade.) → ADR-0012 / 0013.
2. **No-arbiter fallback (A.2):** deterministic **return-to-client** floor only, or also an optional **multisig emergency arbiter**? → ADR-0014.
3. **Spent-`$DSWP` destination (C.3) — legal-sensitive:** auto-burn-on-spend (recommended) vs treasury vs (reject) holder-redistribution. → ADR-0016 + counsel.
4. **Boost market (C.2):** fixed-price start (recommended) vs auction; multiplier cap + reputation-floor values; promoted-slots-per-page cap. → ADR-0016.
5. **Arbiter / juror stake-to-be-eligible (C.4) — legal-sensitive:** is stake-to-work acceptable framing, or keep arbiters unstaked-appointed longer? → ADR-0013 + counsel.
6. **Verification economics (D.2):** confirm rejecting "subject pays verifier + platform takes a cut" in favour of **stake-backed slash-on-fraud**; whether to build the peer-attestation tier at all vs rely on the on-chain-work-history credential. → ADR-0018.
7. **Messaging stack (B.1):** Supabase off-chain + optional E2E; how much (if any) to anchor on-chain. → ADR-0017.

---

## Evidence basis & honesty note

- **Restated from shipped / decided (not inference):** A0 arbiter baseline ([`ADR-0003`](adr/ADR-0003-arbiter-hardening.md) + the verified `DevSwapEscrowV2_6` source); the 97 / 1.5 / 1.5 split + buyback-burn + securities-safe + validate-first ([`TOKENOMICS.md`](TOKENOMICS.md)); non-custodial positioning ([`ADR-0006`](adr/ADR-0006-non-custodial-positioning.md)); the existing wallet ↔ GitHub binding + on-chain reputation counters; `commitmentsAbandoned` ([`ADR-0009`](adr/ADR-0009-funding-trigger.md)).
- **Design inference / engineering judgment (labelled in-text):** all of Areas A.1 – A.5, B, C, D, E recommendations; every numeric (deposits, multiplier caps, timeouts, slot caps, fee bps) is **illustrative**, to be set per-ADR with tests.
- **External protocols** (Kleros, UMA, EAS, Human Passport, World ID, Braintrust, TalentLayer, BrightID, PoH) are referenced as well-known mechanism categories for orientation — **not** cited specifics; any integration requires its own up-to-date due-diligence + audit. Competitor practices are referenced as category exemplars only.
- **The hard truths.** (1) Non-custodial means the protocol cannot fully prevent disintermediation — retention is carrots + reputation, not coercion. (2) Arbitration is the main residual centralisation; full trustlessness is a journey. (3) Any token-demand mechanic carries securities exposure — consumptive design + no-yield framing + counsel is the mitigation, not a guarantee.
- **Status.** Analysis + plan only. No contract / web code changed. Each phase is governance-ratified (its ADR) and audit-gated; mainnet remains behind the audit + multisig + timelock + LP-lock gates.
