# DevSwap Whitepaper: The Decentralized Protocol for Software Services

**Version:** V2.2 · **Date:** May 2026  
**Network:** BNB Smart Chain (BSC) · **Settlement:** USDT · **Utility Token:** `$DSWP`  
**Status:** Testnet — Mainnet gated by independent audit + multisig + timelock

> Full bilingual version (English + Arabic): see [`/whitepaper-ar-en.md`](../whitepaper-ar-en.md) (gitignored — private).

---

## Regulatory Notice

This document is provided for technical and informational purposes only. It does not constitute an offer to sell securities, investment advice, or any guarantee regarding future price, liquidity, or value of `$DSWP`. `$DSWP` is a protocol utility token — not a payment asset, not an investment instrument. The smart contract — not DevSwap — controls all fund flows. No custody. No guarantees.

---

## 1. Executive Summary

DevSwap is a decentralized, non-custodial peer-to-peer marketplace for software development services on BNB Smart Chain. It replaces platform-controlled custody, opaque fees, and centralized dispute resolution with transparent smart contracts, stable USDT settlement, and verifiable on-chain reputation.

**Core thesis:** users should not need to trust a marketplace more. They should be able to verify more.

Every fund flow is controlled by `DevSwapEscrowV2_2` — not DevSwap operators. Developers receive 97% of every released milestone directly on-chain. 1.5% is automatically routed by the contract to buy back and permanently burn `$DSWP` — linking token supply reduction to actual protocol usage (GMV).

---

## 2. The Problem

| Problem | Traditional Platforms | DevSwap |
|---------|----------------------|---------|
| Fees | 10–20% | 3% total (contract-enforced) |
| Fund custody | Platform | Smart contract |
| Payout timing | Days after approval | Same transaction as release |
| Reputation | Platform-owned | On-chain, portable |
| Disputes | Platform decides | Rule-based arbitration |
| Bid access | Pay-to-bid | Free to propose |

---

## 3. Smart Contract: `DevSwapEscrowV2_2`

Key innovations (ADR-0009, ADR-0012, ADR-0014):

- **Hybrid funding:** Full / 10%-deposit / OnAccept modes
- **No-sniping:** Client designates developer at job creation
- **Dispute deposit (V2.4 — DYNAMIC):** `Dispute Deposit = clamp(milestone × 3%, [15 USDT, 150 USDT])`. Loser-funded; split equally among the **majority arbiters** of a 3-panel; discourages frivolous disputes and scales with stake at risk. Replaces V2.2's flat 5 USDT.
- **3-arbiter panel (V2.4):** At `raiseDispute`, the escrow draws **3 unique arbiters** from `DevSwapArbiterPool` via weighted-random selection (entropy: block-hash × jobId × raiser × contract). **2/3 consensus** auto-finalizes; a 7-day voting window then permissionless `finalizeDispute` (tie → client, conservative).
- **Pull-payment refunds:** Winners + majority arbiters claim their funds via `claimWinnerDeposit` / `claimArbiterReward` (CEI strict; nonReentrant). No push-payment DoS surface.
- **Timeout fallback:** After 30 days without resolution, any party triggers `timeoutDispute()` → funds return to client (V2.2 escape hatch, inherited).
- **Arbiter timelock:** 48-hour queue for new arbiter additions in the legacy single-arbiter registry; the staking pool replaces this entirely for V2.4+ disputes.
- **Security:** CEI · ReentrancyGuard · Pausable · Ownable2Step · SafeERC20

**Settlement flow:**
```
Client funds USDT → Smart contract holds → Developer delivers
                          ↓
         Client releases OR timeout triggers auto-release
                          ↓
  Developer: 97% USDT (immediate, same tx)
  Platform:  1.5% USDT
  Protocol:  1.5% USDT → PancakeSwap → $DSWP bought → burned
```

---

## 4. Token Economics: `$DSWP`

### Supply
- **Max Supply:** 100,000,000 DSWP (ERC20Capped — immutable hard ceiling)
- **Mainnet contract:** `0xE950eb93aCa1f29848f5cBac61d78657e3c97287`
- **Burnable:** Yes — supply permanently decreases with every protocol settlement
- **No staking. No yield. No farming. No inflation.**

### Buyback-Burn
On every released milestone: 1.5% USDT → PancakeSwap V2 → `$DSWP` purchased → `burn()` called immediately. If swap fails: deferred to `buybackReserve` (developer payment is never held back).

### Distribution

| Bucket | % | DSWP | Lock |
|--------|---|------|------|
| Strategic Reserve | 5% | 5,000,000 | Locked pending P5 gates (no active offering) |
| PancakeSwap LP | 10% | 10,000,000 | Reserved — deploys only after P5 audit + counsel |
| Team & Founding | 15% | 15,000,000 | 12mo cliff + 24mo linear |
| Ecosystem / Grants | 20% | 20,000,000 | On-chain milestone vesting |
| Airdrop / Community | 10% | 10,000,000 | Post-mainnet snapshot |
| Treasury / Operations | 20% | 20,000,000 | Multisig 3-of-5 + 48h timelock |
| Future Rounds | 15% | 15,000,000 | Reserved, not yet minted |
| Buyback-Burn Reserve | 5% | 5,000,000 | Auto-burned by protocol |

---

## 5. Security Posture & Mainnet Gate (P5)

| Gate | Status |
|------|--------|
| Independent audit (PeckShield / CertiK) | ⏳ pending (owner action) |
| Mythril symbolic execution — CI hard gate | ✅ wired |
| Slither pattern analysis (no medium+ findings) | ✅ green |
| Forge coverage — Escrow + Token 100% | ✅ green |
| Multisig 3-of-5 (Gnosis Safe + Ledger HW) | ⏳ owner action |
| 48–72h Zodiac timelock on parameter changes | ⏳ owner action |
| Qualified-counsel review | ⏳ owner action |
| Geo-block US + OFAC (HTTP 451 on funding + token routes) | ✅ live |
| Token-disclosure interstitial (`/[locale]/token-disclosure`) | ✅ live |

**Strategy rationale (decision 2026-05-26):** the path to mainnet is gated by **independent verification, not a token sale**. The previously documented Fjord LBP has been cancelled; no offering is active and none is committed to. The $DSWP token contract on mainnet (`0x52a68C09f3237B4CB0944F58Ed1CA110a49bE1d9`) exists only for the buyback-burn mechanism and (future) governance. Any future launch is a separate decision pending counsel review and market conditions.

---

## 6. Security

| Layer | Detail |
|-------|--------|
| Language | Solidity 0.8.24, EVM: shanghai |
| Libraries | OpenZeppelin v5 (vendored) |
| Tests | 79+ · Unit/Fuzz(10k)/Invariant/Reentrancy/Fork |
| Coverage | ≥95% lines, 100% functions (Escrow) |
| Static analysis | Slither: 0 high / 0 medium |
| **Audit gate** | **Independent audit required before mainnet** |
| Governance gate | Multisig 3-of-5 + 48h timelock before mainnet |

---

## 7. Roadmap

| Phase | Status | Milestone |
|-------|--------|-----------|
| P1 — Contracts | ✅ Done | V2.2 · 79 tests · 95%+ cov · Slither clean |
| P2 — Frontend | ✅ Done | Next.js 14 · en/ar RTL · All lifecycle flows |
| P3 — Disputes + Index | ✅ Done | /admin · The Graph v0.0.1 live |
| P4 — Economy | ⏸ Deferred 2026-05-26 | Token launch path withdrawn pending P5 audit + counsel; buyback-burn live in V2.2 |
| P5 — Mainnet | ⛔ Gated | Audit + Multisig + Legal + Niche launch |
| P6 — Governance | 📋 Planned | ERC20Votes → GovernorDevSwap → TimelockControllers |

---

## 8. Legal Notice

This whitepaper is informational only. It does not constitute an offer to sell securities, investment advice, or any guarantee of future value. `$DSWP` is a protocol utility token. The smart contract — not DevSwap — controls all fund flows. No custody. No guarantees. No refunds from a central party. Any mainnet deployment will be preceded by independent security audit, multisig governance, and locked liquidity.

---

*DevSwap Protocol · devswap.pro · dev@devswap.pro*  
*Token Contract (BSC Mainnet): `0xE950eb93aCa1f29848f5cBac61d78657e3c97287`*  
*Escrow (BSC Testnet): `0xCEE07220dEC813f8A58b7Da73349dabbc4005840`*
