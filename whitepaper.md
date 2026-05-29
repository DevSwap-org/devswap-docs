# DevSwap Whitepaper — The On-Chain Protocol for Solidity & Web3 Talent

**Version:** V2.6 · **Date:** May 2026
**Network:** BNB Smart Chain (BSC) · **Settlement:** USDT · **Utility Token:** `$DSWP`
**Status:** Testnet (live arbiter pool, dispute panel ready) — mainnet gated by independent audit, multisig, and timelock

---

## Regulatory Notice

This document is provided for technical and informational purposes only. It is not an offer to sell securities, investment advice, or any guarantee regarding future price, liquidity, or value of `$DSWP`. `$DSWP` is a protocol utility token — not a payment asset, not an investment instrument. The smart contract — not the protocol team — controls all fund flows. No custody. No guarantees.

---

## 1. Executive Summary

DevSwap is a non-custodial, peer-to-peer marketplace **built for Solidity developers, smart-contract auditors, and Web3 builders** on BNB Smart Chain. It replaces platform-controlled custody, opaque fees, and centralized dispute resolution with transparent smart contracts, stable USDT settlement, and verifiable on-chain reputation. A parallel Arabic-language track makes the same primitives available to the MENA developer community in RTL with a hand-translated UI.

**Core thesis:** users should not need to trust a marketplace more — they should be able to verify more.

Every fund flow is controlled by `DevSwapEscrowV2_6` — not by DevSwap operators. Developers receive **97%** of every released milestone directly on-chain. **1.5%** is routed automatically by the contract to buy back and permanently burn `$DSWP` — linking token-supply reduction to actual protocol usage (GMV).

---

## 2. The Problem

| Problem | Traditional Platforms | DevSwap |
|---------|----------------------|---------|
| Fees | 10 – 20 % | 3 % total (contract-enforced) |
| Fund custody | Platform | Smart contract |
| Payout timing | Days after approval | Same transaction as release |
| Reputation | Platform-owned | On-chain, portable |
| Disputes | Platform decides | Rule-based arbitration by a staked panel |
| Bid access | Pay-to-bid | Free to propose |

---

## 3. Smart Contract: `DevSwapEscrowV2_6`

Key innovations:

- **Hybrid funding** — Full / 10%-deposit / OnAccept modes match how the work actually starts.
- **No-sniping** — the client designates the developer at job creation.
- **Symmetric 3% dispute bond** — both client and developer post the same bond at `raiseDispute`. The loser forfeits; the winner is refunded.
- **4-way split on resolution** — 50 % to the winner / 35 % to the majority panel / 10 % to buyback-burn / 5 % to the protocol fee bucket.
- **3-arbiter panel** — at `raiseDispute` the escrow draws 3 unique arbiters from `DevSwapArbiterPool` via weighted-random selection (entropy: block-hash × jobId × raiser × contract). 2/3 consensus auto-finalizes; a 7-day voting window then permits permissionless `finalizeDispute` (tie → client, conservative).
- **Pull-payment refunds** — winners and majority arbiters claim their funds via `claimWinnerDeposit` / `claimArbiterReward` (strict CEI, `nonReentrant`). No push-payment DoS surface.
- **Timeout fallback** — after 30 days without resolution, any party may trigger `timeoutDispute()` and funds return to the client.
- **Security primitives** — CEI · `ReentrancyGuard` · `Pausable` · `Ownable2Step` · `SafeERC20`.

**Settlement flow:**
```
Client funds USDT  →  Smart contract holds  →  Developer delivers
                              ↓
              Client releases  OR  timeout triggers auto-release
                              ↓
        Developer  : 97  % USDT  (immediate, same tx)
        Protocol   : 1.5% USDT  (platform fee)
        Protocol   : 1.5% USDT  →  PancakeSwap  →  $DSWP bought  →  burned
```

---

## 4. Token Economics: `$DSWP`

### Supply
- **Max supply** — 100,000,000 DSWP (`ERC20Capped` — immutable hard ceiling)
- **Mainnet contract** — `0x52a68C09f3237B4CB0944F58Ed1CA110a49bE1d9`
- **Burnable** — yes; supply permanently decreases with every protocol settlement
- **No staking. No yield. No farming. No inflation.**

### Buyback-Burn
On every released milestone, 1.5 % of USDT is routed to PancakeSwap V2 to purchase `$DSWP`, which is then burned via `burn()`. If the swap fails, the amount defers to `buybackReserve` for a later batch burn — the developer payment is never held back.

### Distribution

| Bucket | % | DSWP | Lock |
|--------|---|------|------|
| Strategic Reserve | 5 % | 5,000,000 | Locked pending mainnet gates (no active offering) |
| PancakeSwap LP | 10 % | 10,000,000 | Reserved — deploys only after audit + counsel review |
| Team & Founding | 15 % | 15,000,000 | 12-month cliff + 24-month linear |
| Ecosystem / Grants | 20 % | 20,000,000 | On-chain milestone vesting |
| Airdrop / Community | 10 % | 10,000,000 | Post-mainnet snapshot |
| Treasury / Operations | 20 % | 20,000,000 | Multisig 3-of-5 + 48 h timelock |
| Future Rounds | 15 % | 15,000,000 | Reserved, not yet minted |
| Buyback-Burn Reserve | 5 % | 5,000,000 | Auto-burned by protocol |

---

## 5. Security Posture & Mainnet Gate

| Gate | Status |
|------|--------|
| Independent audit (PeckShield / CertiK) | Pending |
| Mythril symbolic execution — CI hard gate | Live |
| Slither pattern analysis (no medium+ findings) | Green |
| Forge coverage — Escrow & Token ≥ 95 % | Green |
| Multisig 3-of-5 (Gnosis Safe + Ledger HW) | Pending |
| 48 – 72 h Zodiac timelock on parameter changes | Pending |
| Qualified-counsel review | Pending |
| Geo-block US + OFAC (HTTP 451 on funding + token routes) | Live |
| Token-disclosure interstitial (`/[locale]/token-disclosure`) | Live |

**Path to mainnet.** Mainnet is gated by independent verification, not by a token sale. No public offering is active. The `$DSWP` token contract on mainnet exists only for buyback-burn settlement and future on-chain governance. Any future offering is a separate decision pending counsel review and market conditions.

---

## 6. Security Summary

| Layer | Detail |
|-------|--------|
| Language | Solidity 0.8.34, EVM target: shanghai |
| Libraries | OpenZeppelin v5 (vendored) |
| Tests | 412 (20 suites · unit · fuzz @ 10 k · invariant · reentrancy · mainnet-fork) |
| Coverage | Escrow ≥ 95 % lines; functions 100 % |
| Static analysis | Slither: 0 high / 0 medium |
| **Audit gate** | **Independent audit required before mainnet** |
| Governance gate | Multisig 3-of-5 + 48 h timelock required before mainnet |

---

## 7. Roadmap

A live, dateless roadmap (Past · Now · Next) is published at [`ROADMAP.md`](./ROADMAP.md). High-level phase view:

| Phase | Status | Milestone |
|-------|--------|-----------|
| P1 — Contracts | ✅ Done | V2.6 · 412 tests · ≥ 95 % cov · Slither clean |
| P2 — Frontend | ✅ Done | Next.js 14 · en / ar RTL · full lifecycle flows |
| P3 — Disputes + Index | ✅ Done | Staked arbiter pool (12 active arbiters live) · keeper service running · subgraph live |
| P4 — Economy | Deferred | Public offering path withdrawn pending audit + counsel; buyback-burn live |
| P5 — Mainnet | Gated | Audit + multisig + counsel + niche launch |
| P6 — Governance | Planned | `ERC20Votes` → `GovernorDevSwap` → `TimelockController` |

---

## 8. Legal Notice

This whitepaper is informational only. It is not an offer to sell securities, investment advice, or any guarantee of future value. `$DSWP` is a protocol utility token. The smart contract — not the protocol team — controls all fund flows. No custody. No guarantees. No refunds from a central party. Any mainnet deployment will be preceded by independent security audit, multisig governance, and locked liquidity.

---

*DevSwap Protocol · [devswap.pro](https://devswap.pro) · contact@devswap.pro*
*Token contract (BSC mainnet): `0x52a68C09f3237B4CB0944F58Ed1CA110a49bE1d9`*
*Escrow (BSC testnet): `0x22633bd98d6F9AD4dF499b77429459F5574B4dFe`*
