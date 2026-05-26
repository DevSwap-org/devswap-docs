# DevSwap Protocol — Documentation

> A **Security-Hardened Protocol** for on-chain software-services settlement. Funds are locked in a
> verified smart contract, released only on the client's explicit approval (or via deterministic
> on-chain dispute and timeout rules). Non-custodial by design; no middleman, no platform escrow.

- 🌐 Application: **https://devswap.pro**
- 📜 Contracts (public source): **https://github.com/DevSwap-org/devswap-contracts**
- 🔐 Security policy: **https://github.com/DevSwap-org/.github/blob/main/SECURITY.md**
- 🛡️ Full audit notes: [SECURITY-AUDIT.md](SECURITY-AUDIT.md)
- 🏛️ Governance roadmap: [GOVERNANCE.md](GOVERNANCE.md)

---

## 1. Protocol Security — first principle

DevSwap Protocol's path to a production deployment is gated by **independent verification, not by
any token offering**. The protocol is operating in testnet today; the public mainnet release is
gated by the **seven Security Gates** documented below.

| # | Gate | Type | Status |
|---|---|---|---|
| 1 | Symbolic execution (Mythril) — CI hard gate | Automated | ✅ wired |
| 2 | Pattern analysis (Slither) — 0 high · 0 medium findings | Automated | ✅ green |
| 3 | Test coverage — Escrow + Token 100% lines / statements / branches / functions | Automated | ✅ green |
| 4 | Test suite — unit · fuzz @ 10,000 runs · invariant · mainnet-fork (79+ tests) | Automated | ✅ green |
| 5 | Independent third-party audit (e.g., PeckShield / CertiK) | Manual | ⏳ pending |
| 6 | Multisig 3-of-5 (hardware wallets) + Zodiac timelock (48–72h) | Manual | ⏳ pending |
| 7 | Qualified-counsel review of disclosure copy + jurisdiction posture | Manual | ⏳ pending |

The four automated gates are enforced on every commit to `main` in the public contracts repository.
The three manual gates are protocol-maintainer actions required before any mainnet launch.

---

## 2. P0 + P1 Legal-Safety Controls — live today

Two layers of protocol-level safety controls are in force today across every DevSwap Protocol
surface — these are documented in detail in [SECURITY-AUDIT §9](SECURITY-AUDIT.md):

- **Geo-restriction (RFC 7725 / HTTP 451).** Requests from United States persons and from
  OFAC-sanctioned jurisdictions (Cuba, Iran, North Korea, Syria, Russia, Belarus) are blocked
  from funding routes and any token-related disclosure pages, returning an inline bilingual
  status-451 response.
- **Server-rendered disclosure interstitial.** Any external link referencing the protocol utility
  token is routed through a server-rendered page with a strict host allowlist (prevents
  open-redirect abuse) and an explicit utility-token disclosure in both supported locales.

---

## 3. How the settlement protocol works

| Step | Actor | Effect |
|------|-------|--------|
| 1. Lock | Client | Creates a task and funds it; USDT is locked in the smart contract. |
| 2. Accept | Developer | Designated developer accepts the task and (later) submits the deliverable. |
| 3. Release | Client | Approves the delivery → the smart contract releases 97% to the developer atomically. |
| 4. Dispute / timeout | Either party | Transparent on-chain dispute path; deterministic timeout fallback. |

Task lifecycle states: `Open → Accepted → Submitted → Released` (terminal) ·
`Cancelled` (returns funds to client) · `Disputed` (resolved by arbitration policy).

### 3.1 Economics — contract-enforced, not policy

| Share | Recipient |
|---:|---|
| **97%** | Developer |
| **1.5%** | Protocol operations |
| **1.5%** | Buyback-and-burn of the protocol utility token |

Total fee: **3%**, enforced in contract bytecode (the 97% developer floor and 3% total ceiling are
**immutable** — no governance vote can change them; see [GOVERNANCE.md §5.4](GOVERNANCE.md)). The
buyback-and-burn segment swaps to the protocol utility token and burns it; a failed swap is deferred
to a reserve for later bulk burn, so a developer's payout is **never** blocked by market conditions.

---

## 4. Protocol Utility Token

The protocol utility token is a capped ERC-20 (max supply: 100,000,000) used exclusively by the
buyback-and-burn mechanism and (in the future) by decentralized governance. It is **not** a payment
asset, not an investment instrument, and not a claim on protocol revenue.

**No price, yield, dividend, revenue share, or return is promised.** There is no active token sale
on or facilitated by the DevSwap Protocol interface; any prior third-party offering surface has
been removed pending qualified-counsel review.

The verified token contract address is available on the project's official BscScan listing, surfaced
from the application only through the server-rendered disclosure interstitial described in §2.

---

## 5. Decentralized Governance Roadmap

DevSwap Protocol's long-term goal is to be **ungovernable by any single party**, including its
original contributors. The transition follows four phases (G0 → G4), documented in detail in
[GOVERNANCE.md](GOVERNANCE.md):

| Phase | What | Status |
|-------|------|--------|
| G0 | Owner-controlled testnet — current state | Today |
| G1 | Mainnet launch behind 3-of-5 multisig + 48h timelock | Pending Security Gates 5 + 6 |
| G2 | Protocol utility token gains `ERC20Votes` (snapshot-based voting power) | Future |
| G3 | DAO Governor + three-tier timelock (48h / 7d / 14d) controls parameter changes | Future |
| G4 | Guardian-veto sunset; no single entity has special on-chain power | ≥ 12 months post-G3, by community supermajority |

Several **immutable floors** are enforced at the Solidity level — no governance vote, even at
maximum supermajority, can change them: the ≥ 97% developer share, the ≤ 3% total fee ceiling, the
ReentrancyGuard pattern on every state-mutating function, and the per-dispute snapshot rule that
closes the puppet-arbiter attack class. Full list in [GOVERNANCE.md §5.4](GOVERNANCE.md).

---

## 6. Technical Stack — public references

- **Smart contracts:** Solidity 0.8.24 (Foundry), OpenZeppelin v5 (vendored — reproducible builds).
- **Settlement asset:** USDT on BNB Smart Chain (BEP-20, 18 decimals — distinct from Ethereum's 6).
- **DEX integration:** PancakeSwap V2 router (for the buyback-and-burn mechanism).
- **Network identifiers:** BNB Smart Chain mainnet `chainId 56` · testnet `chainId 97`.
- **Application stack:** Next.js on Cloudflare Workers (OpenNext), wagmi / viem, RainbowKit, IPFS
  metadata, bilingual EN / AR with full RTL support.

The contracts repository (`devswap-contracts`) is public and verified on BscScan. Reproducible
builds: the OpenZeppelin v5 dependency is vendored at a pinned commit hash.

---

## 7. Trust & Verification — what to check

- **Non-custodial:** the smart contract is the actor that moves funds — not the protocol operators.
- **Verifiable on-chain:** every fund flow is an on-chain event with a transaction hash; the
  subgraph indexes every state transition for transparent third-party verification.
- **Reproducible audit:** tooling outputs (forge coverage, Slither, Mythril CI) are reproducible
  from the public contracts repository.
- **Public verification:** the contracts are source-verified on BscScan; readers should consult the
  verified source directly rather than trusting any link.

---

## 8. FAQ

**Do I need an account to browse?** No — read-only browsing is open globally; connect a wallet only
when you want to fund or accept work (these routes are subject to the §2 geo-restrictions).

**Who holds my funds?** The smart contract, until your explicit approval or until a dispute /
timeout rule fires. The protocol operators have no key that can move client funds outside the
documented contract paths.

**What are the fees?** 3% total, split 1.5% protocol operations + 1.5% buyback-and-burn; the
developer receives 97%. These shares are immutable in contract bytecode (see §3.1).

**Which networks?** Marketplace is live on BNB Smart Chain **testnet** (chainId 97). Mainnet
deployment is gated by the seven Security Gates in §1.

**Is there a token sale?** **No.** See §4 and [SECURITY-AUDIT §9](SECURITY-AUDIT.md).

**Where can I read the security audit?** Right here: [SECURITY-AUDIT.md](SECURITY-AUDIT.md).

---

> **Status:** Security-Hardened Protocol on testnet. Nothing in this document is financial advice
> or a solicitation. License: [MIT](LICENSE).
