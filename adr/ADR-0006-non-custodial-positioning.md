# ADR-0006 — Non-Custodial Protocol Positioning

- **Status:** Accepted
- **Date:** 2026-05-23
- **Supersedes/relates:** complements CLAUDE.md §18 (Legal-Safe Language) and PR L-1 (`fix/legal-safe-language`).

## Context

DevSwap settles work via **on-chain smart contracts** (`DevSwapEscrow` / `DevSwapEscrowV2_1`). The
contract — not the company — holds and releases USDT. Using language that implies **custody**, **payment
processing**, or **financial guarantees** ("we hold your funds", "we pay you", "guaranteed", "insured",
"escrow service") could create regulatory obligations the project does not intend to assume
(money-transmitter / custodian / financial-services / insurance framings vary by jurisdiction —
e.g. SAMA, VARA, and others). It can also be read as a promise the protocol cannot legally make.

A real instance was found and fixed: user-facing copy used custody/guarantee verbs in both EN
and AR (the hero literally read "pay with a guarantee"). See PR L-1.

## Decision

All **user-facing** copy positions DevSwap as a **decentralized protocol**, not a service provider:

- The **smart contract** is the actor that locks and releases funds — *"funds are locked in the smart
  contract", "released automatically upon delivery"* — never *"we hold / we pay / we release"*.
- DevSwap **does not hold, send, guarantee, or insure** funds. It is the **interface** to a publicly
  deployed contract.
- Forbidden/required vocabulary is enforced by CI (`scripts/check-legal-language.sh`).
- The **technical exception** stands: `escrow` may be used in ADRs, technical docs, and variable names
  (`escrowBalance`, `DevSwapEscrowV2_1`), but **never** in user-facing copy.

**Canonical positioning statement** (rendered on `/how-to-start`; home hero is already non-custodial
post-L-1):

> "DevSwap is a decentralized protocol connecting clients and developers through on-chain smart
> contracts. We don't hold funds or process payments. All settlement happens autonomously per the
> publicly deployed smart contract."
>
> (Arabic localization renders the same statement with negated custody verbs — see the public app.)

## Consequences

- **Legal:** reduced regulatory exposure (not eliminated — independent counsel still required before
  mainnet / public launch; this is positioning, not legal advice).
- **UX:** slightly more technical language, offset by clear first-encounter explainer copy (UX-5).
- **Marketing:** strengthens the decentralization moat (DOC-3 / MOAT).
- **Engineering:** code variable names unaffected; only user-facing copy is regulated.
- **Note:** the positioning statement deliberately uses negated forms ("we don't hold"); the
  legal-language CI guard allows negated usage in the positioning key and only flags affirmative
  custodial claims.
