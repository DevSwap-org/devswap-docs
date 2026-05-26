# DevSwap — Documentation

Public documentation for [**DevSwap**](https://devswap.pro), an on‑chain freelance marketplace on
**BNB Smart Chain** where the budget is locked in USDT and released by a smart contract on the
client's approval — no platform custody, no middleman.

- 🌐 App: **https://devswap.pro**
- 📜 Contracts: **https://github.com/DevSwap-org/devswap-contracts**
- 🔐 Security policy: **https://github.com/DevSwap-org/.github/blob/main/SECURITY.md**

---

## How it works
1. **Lock** — the client creates a task and funds it; USDT is locked in the smart contract.
2. **Accept & deliver** — a developer accepts the task and submits the work (proof on IPFS / GitHub).
3. **Release** — the client approves → the contract **releases funds automatically** to the developer.
   If something goes wrong, either party can open a **transparent dispute**; a submit‑timeout protects
   against an unresponsive party.

Task states: `Open → Accepted → Submitted → Released` (terminal) · `Cancelled` (returns funds to the
client) · `Disputed` (frozen pending resolution).

## Economics
On successful release the locked USDT splits, enforced by the contract:

| Share | Recipient |
|---:|---|
| **97%** | Developer |
| **1.5%** | Platform fee |
| **1.5%** | `$DSWP` community **buyback‑and‑burn** |

Total fee: **3%**. The 1.5% buyback swaps to `$DSWP` and burns it (reducing supply); a failed swap is
deferred to a reserve for a later bulk burn, so a developer's payout is **never** blocked.

## `$DSWP` token
ERC‑20, **capped at 100,000,000**, burnable, `Ownable2Step`. A **utility** token used by the
buyback‑and‑burn mechanism. No price, yield, or return is promised — `$DSWP` is not an investment.

### `$DSWP` on BSC mainnet
The `$DSWP` ERC‑20 token contract is deployed on BNB Smart Chain mainnet. The marketplace
itself is still on **testnet** (see the top of this page); the token contract on mainnet is independent
of the marketplace and exists for the buyback‑and‑burn mechanism and (future) community governance.

- **Token contract (mainnet):** [`0x52a68C09f3237B4CB0944F58Ed1CA110a49bE1d9`](https://bscscan.com/token/0x52a68C09f3237B4CB0944F58Ed1CA110a49bE1d9) — BscScan

`$DSWP` is a utility token. No price, yield, revenue share, or return is promised. It is not an
investment product.

**No active token sale.** Any previously linked third‑party sale page has been removed pending
qualified‑counsel review. The DevSwap interface displays a server‑rendered interstitial
disclosure before any external $DSWP‑related link is followed, and US persons + OFAC‑sanctioned
jurisdictions (Cuba, Iran, North Korea, Syria, Russia, Belarus) are geo‑blocked from
token‑related and funding routes (HTTP 451 — see [SECURITY-AUDIT §9](SECURITY-AUDIT.md#9-legal-safety-controls-2026-05-26)).

## Network & technical facts
- **USDT on BSC has 18 decimals** (≠ Ethereum's 6) — amounts are computed in integer wei.
- BSC mainnet `chainId 56` · testnet `chainId 97` · PancakeSwap V2 router for the buyback swap.
- Stack: Solidity 0.8.24 (Foundry) · Next.js 14 on **Cloudflare Workers** (OpenNext) · wagmi/viem ·
  RainbowKit · The Graph subgraph · IPFS metadata · bilingual EN/AR (RTL).

## Trust & safety
- **Non‑custodial**: the smart contract holds and moves funds — not the platform.
- Contracts are tested (unit + fuzz + invariant + mainnet‑fork) and statically analyzed; a third‑party
  audit is required before any mainnet launch with real funds.
- Source is public for verification (`devswap-contracts`) and contracts are verified on BscScan.

## Further documentation

- **[SECURITY-AUDIT.md](SECURITY-AUDIT.md)** — internal security audit notes for the smart contracts
  (Phase 1 + V2.1 + V2.2): tooling results (forge, slither, coverage), findings, and the P5
  pre‑mainnet checklist (independent audit + multisig + timelock + LP lock).
- **[GOVERNANCE.md](GOVERNANCE.md)** — decentralisation & DAO governance roadmap (G0 → G4): what
  the DAO will govern, the three timelock tiers, and the immutable floors no governance vote can
  touch (≥ 97% developer share, ≤ 3% total fee, dispute snapshot rule, etc.).

## FAQ
**Do I need an account to browse?** No — browsing is free; connect a wallet only to fund or accept work.
**Who holds my money?** The smart contract, until you approve the delivery. The platform cannot move it.
**What are the fees?** 3% total (1.5% platform + 1.5% buyback‑burn); the developer receives 97%.
**Which networks?** Marketplace live on BNB Smart Chain **testnet** today; `$DSWP` token deployed on **mainnet**.

---

> Status: marketplace on **testnet**, `$DSWP` token on **mainnet**. Nothing here is financial advice.
> License: [MIT](LICENSE).
