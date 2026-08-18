# Fees and third-party costs

**Product:** Smart Wallet (Chrome / Opera MV3)  
**Last updated:** 2026-08-16  
**Applies to:** In-wallet (internal) Swap and Bridge. External DEX/bridge sites are separate.

This page summarizes **Smart Wallet platform fees** and **current third-party service fees** that can appear on the same atomic transaction. It is not a complete list of every on-chain cost (network gas, DEX impact, relayer fees).

Smart Wallet rates are fixed product constants:

- Internal swap: **0.45%** (45 bps)
- Internal bridge: **0.85%** (85 bps)

LI.FI’s service fee is **quote-derived**. It is **currently** **0.25%** (25 bps) on LiFi routes. The wallet displays the verified fee returned with the route. Do not treat 0.25% as permanent.

---

## SMART WALLET INTERNAL SOLANA SWAPS

Smart Wallet charges **0.45%** on eligible internal Jupiter swaps. The fee is included atomically in the same Solana transaction. If the fee-bearing transaction is rejected or fails, neither the swap nor Smart Wallet fee completes. Solana network fees and market effects are separate.

There is **no** LI.FI service fee on Jupiter. Combined service/platform total: **0.45%**.

## SMART WALLET INTERNAL EVM SWAPS

Smart Wallet charges **0.45%**. LI.FI currently charges a separate **0.25%** service fee, producing a current combined service/platform total of **0.70%**. Network gas, DEX/tool costs, liquidity effects, price impact and slippage are additional or reflected separately in the quote. The Smart Wallet and LI.FI fees are included in the same atomic source-chain transaction.

Smart Wallet does **not** receive LI.FI’s 0.25%.

## SMART WALLET INTERNAL BRIDGES

Smart Wallet charges **0.85%**. LI.FI currently charges a separate **0.25%** service fee, producing a current combined service/platform total of **1.10%**. Source-chain gas and bridge/relayer/tool costs are separate. The Smart Wallet and LI.FI fees are included atomically in the source-chain transaction.

Destination completion does not charge a second Smart Wallet fee.

## EXTERNAL DEXES AND BRIDGES

Smart Wallet does not charge its internal platform fee when a user leaves Smart Wallet and uses an external DEX or bridge directly. External providers and networks may charge their own fees.

## GENERAL

Fee amounts and estimated outputs are shown before confirmation. Third-party fees can change. The wallet displays the verified current provider fee returned with the route. Users should review the complete transaction summary before signing.

Combined service/platform fees **exclude** network gas, liquidity effects, price impact and bridge/relayer costs.

Failed or rejected fee-bearing transactions complete neither the main action nor the Smart Wallet fee.

---

## Summary table

| Action | Smart Wallet | LI.FI service (currently) | Combined service/platform | Separate |
|--------|--------------|---------------------------|---------------------------|----------|
| Internal Jupiter swap | 0.45% | none | **0.45%** | Solana network fee, price impact |
| Internal LiFi EVM swap | 0.45% | 0.25% | **0.70%** | Gas, DEX/tool, impact, slippage |
| Internal LiFi EVM-source bridge | 0.85% | 0.25% | **1.10%** | Source gas, relayer/tool, dest gas |
| Send / Receive / Buy (Onramper) | 0% | — | 0% | Network / Onramper |
| External DEX or bridge site | 0% | — | 0% | That site + network |

---

## Rounding

Smart Wallet fees use integer base units:

- Swap: `floor(grossInput × 45 / 10000)`
- Bridge: `floor(grossInput × 85 / 10000)`

Very small inputs can floor to zero; those routes fail closed rather than sending a fee-free internal trade.

LI.FI’s amount is taken from the verified quote (`feeSplit.lifiFee`) when present.

Quoted output from Jupiter/LiFi is already after encoded source-side fees. The UI does not subtract those fees again from the displayed receive amount.

---

## Where fees appear in the app

| Surface | What you see |
|---------|----------------|
| **Swap** panel | Smart Wallet %, current LI.FI service % on EVM, combined total, exclude note |
| **Bridge** panel | Smart Wallet 0.85%, current LI.FI service %, combined 1.10% when LI.FI is 25 bps |
| **Settings** | Link to this **Fee disclosure** page |

---

## Contact

- **Support:** [smartwallethelp@outlook.com](mailto:smartwallethelp@outlook.com)
- **Developer:** [Whitebrick86@gmail.com](mailto:Whitebrick86@gmail.com)
- **Contacts page:** [CONTACTS.md](./CONTACTS.md)

---

## Related docs

| File | Role |
|------|------|
| [PRIVACY-POLICY.md](./PRIVACY-POLICY.md) | Privacy |
| [CONTACTS.md](./CONTACTS.md) | Contact emails |
| [DOCUMENTATION.txt](../DOCUMENTATION.txt) | Full user guide |
| [ARCHITECTURE.md](../ARCHITECTURE.md) | Product architecture |

---

*Not financial advice. Cryptocurrency involves risk of permanent loss of funds. Always verify amounts before approving.*
