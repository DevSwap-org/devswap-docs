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

## FAQ
**Do I need an account to browse?** No — browsing is free; connect a wallet only to fund or accept work.
**Who holds my money?** The smart contract, until you approve the delivery. The platform cannot move it.
**What are the fees?** 3% total (1.5% platform + 1.5% buyback‑burn); the developer receives 97%.
**Which networks?** Live on BNB Smart Chain **testnet** today.

---

> Status: **testnet**. Nothing here is financial advice. License: [MIT](LICENSE).
