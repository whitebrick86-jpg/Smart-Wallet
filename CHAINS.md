# Networks (including Arbitrum, Optimism, Avalanche)

**Product:** Smart Wallet  
**Docs snapshot:** network-add **0.11.159** · **Live product:** **0.11.257** ([PRODUCT.md](./PRODUCT.md))  
**Last updated:** 2026-08-13  
**Repository:** Documentation only — extension source is **not** published here.

This is the account of **which networks the wallet supports**, how they are ordered, and what was added for **Arbitrum One**, **Optimism**, and **Avalanche C-Chain** (0.11.156–0.11.159).

Loads (ping / HTTPS / RPC counts): **[LOADS.md](./LOADS.md)**.  
Internal swap path: **[INTERNAL-DEX.md](./INTERNAL-DEX.md)**.

---

## 1. Canonical order (Networks panel = Bridge)

One source: `chain-registry.js` → `displayChains()` (Solana first, then registry order).

Both Bridge dropdowns use `bridgeableChains()` from that same list. Bitcoin and Sui are **not** bridgeable. The destination list **drops the selected source** and does **not** reorder the rest.

| # | id | Name | Kind | Native | chainId | Bridge |
|---|----|------|------|--------|---------|--------|
| 1 | `solana` | Solana | solana | SOL | — | Yes |
| 2 | `ethereum` | Ethereum | evm | ETH | 1 | Yes |
| 3 | `bitcoin` | Bitcoin | bitcoin | BTC | — | No |
| 4 | `polygon` | Polygon | evm | POL | 137 | Yes |
| 5 | `sui` | Sui | sui | SUI | — | No |
| 6 | `robinhood` | Robinhood Chain | evm | ETH | 4663 | Yes |
| 7 | `base` | Ethereum Base | evm | ETH | 8453 | Yes |
| 8 | `bsc` | BNB Smart Chain | evm | BNB | 56 | Yes |
| 9 | `arbitrum` | **Arbitrum One** | evm | ETH | **42161** | Yes |
| 10 | `optimism` | **Optimism** | evm | ETH | **10** | Yes |
| 11 | `avalanche` | **Avalanche C-Chain** | evm | AVAX | **43114** | Yes |

Regression: `tools/test-chain-order-bridge.js`.

---

## 2. Arbitrum One (0.11.156, hooks 0.11.159)

| | |
|--|--|
| chainId / gas | **42161** · **ETH** (L2, same fee class as Base) |
| Official RPC | `https://arb1.arbitrum.io/rpc` |
| Live-good RPCs | official, publicnode, publicnode-rpc, drpc, meowrpc |
| Removed after probe | `arbitrum.llamarpc.com` (DNS fail) |
| Explorer | `https://arbiscan.io` · Blockscout `https://arbitrum.blockscout.com` |
| WETH | `0x82aF49447D8a07e3bd95BD0d56f35241523fBab1` |
| Native USDC | `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` — Circle |
| USDC.e | `0xFF970A61A04b1cA14834A43f5dE4533eBDDB5CC8` — Circle (bridged, not Circle-issued) |
| Uniswap V3 SwapRouter02 | `0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45` — Uniswap docs (recorded; native V3 encode not wired) |
| Internal DEX | **LiFi first.** 0x is listed but **skipped** at runtime (`no_backend`). |
| Bridge | LiFi source and dest |
| dApp | `wallet_switchEthereumChain` / shared approve / capabilities `0xa4b1` |
| Live verified | `eth_chainId` 42161 · USDC decimals **6** · LiFi lists chain · LiFi quote ETH→USDC HTTP 200 |
| Not live-certified | In-wallet Send click, Ledger HID, Uniswap tab, completed bridge |

---

## 3. Optimism / OP Mainnet (0.11.157, hooks 0.11.159)

| | |
|--|--|
| chainId / gas | **10** · **ETH** (L2, same fee class as Base) |
| Official RPC | `https://mainnet.optimism.io` |
| Live-good RPCs | official, publicnode, publicnode-rpc, drpc, 1rpc |
| Removed after probe | `optimism.meowrpc.com` (DNS fail) |
| Explorer | `https://optimistic.etherscan.io` · Blockscout `https://optimism.blockscout.com` |
| WETH | `0x4200000000000000000000000000000000000006` — OP-stack **predeploy** (same address as Base **by OP design**, not a copied USDC) |
| Native USDC | `0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85` — Circle |
| USDC.e | `0x7F5c764cBc14f9669B88837ca1490cCa17c31607` — Optimism token list |
| Uniswap V3 SwapRouter02 | `0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45` — Uniswap docs |
| Internal DEX | **LiFi first.** 0x skipped at runtime. |
| Bridge | LiFi source and dest |
| dApp | shared approve / capabilities `0xa` |
| Live verified | `eth_chainId` 10 · USDC decimals **6** · LiFi lists OP Mainnet · LiFi quote ETH→USDC HTTP 200 |
| Not live-certified | In-wallet Send / Ledger / Uniswap tab / completed bridge |

---

## 4. Avalanche C-Chain (0.11.158, hooks 0.11.159)

| | |
|--|--|
| chainId / gas | **43114** · **AVAX** (not an ETH L2; own floors / CoinGecko `avalanche-2`) |
| Official RPC | `https://api.avax.network/ext/bc/C/rpc` |
| Live-good RPCs | official, publicnode, drpc, 1rpc |
| Removed after probe | Ankr (HTTP 200, no `chainId`), `avalanche.public-rpc.com` (403) |
| Explorer | `https://snowtrace.io` |
| WAVAX | `0xB31f66AA3C1e785363F0875A1B74E27b85FD66c7` — Uniswap Avalanche deployments |
| Native USDC | `0xB97EF9Ef8734C71904D8002F8b6Bc66Dd9c48a6E` — unique vs ETH / Base / Arb / OP |
| USDC.e | **Not listed** (no Circle-grade source used) |
| Uniswap V3 SwapRouter02 | `0xbb00FF08d01D300023C629E8fFfFcb65A5a578cE` — **different** from ETH/Arb/OP |
| QuoterV2 | `0xbe0F5544EC67e9B3b2D979aaA43f18Fd87E6257F` |
| Internal DEX | **LiFi first.** 0x skipped at runtime. |
| Prices | CoinGecko `avalanche-2` · Binance `AVAXUSDT` (one market WS when this net is active) |
| Bridge | LiFi source and dest |
| dApp | shared approve / capabilities `0xa86a` |
| Live verified | `eth_chainId` 43114 · USDC decimals **6** · LiFi lists Avalanche · LiFi quote AVAX→USDC HTTP 200 |
| Not live-certified | In-wallet Send / Ledger / Uniswap tab / completed bridge |

---

## 5. What 0.11.159 added (Base-template hooks)

Registry-only was not enough. These tables now include the three nets the same way Base is wired:

- Swap/Holdings USDC seed (`CHAIN_USD_STABLE`)
- History explorer + log RPC prefer + lookback budgets
- Send preferred RPC heads and fee summaries
- Shared dApp approve + EIP-1193 capabilities hexes
- Preflight native symbol (AVAX, not “gas”)
- Offscreen / Ledger fee floors and `chainMeta`
- Onramp network profiles
- CoinGecko majors include `avalanche-2` (same HTTPS call, extra id)

---

## 6. Rules that did not change

- Sequential RPC only (no parallel fan-out)
- Sign-once; receipt status 1 / 0 / missing
- Smart Wallet platform fees **45 bps** swap / **85 bps** bridge; LiFi EVM routes currently add **25 bps** service fee (combined **70 / 110 bps**)
- Solana stays Jupiter-only and separate from EVM
- Existing BNB / Base / Polygon / Ethereum / Robinhood / Solana behavior unchanged (regression suite **VERIFY OK**)

---

## Related docs

| File | Role |
|------|------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Whole-wallet build |
| [LOADS.md](./LOADS.md) | Ping / HTTPS / RPC counts including the new nets |
| [INTERNAL-DEX.md](./INTERNAL-DEX.md) | Quote / execute / fee due |
| [BUGS-AND-FIXES.md](./BUGS-AND-FIXES.md) | 0.11.156–159 in Part D |

---

*Not financial advice. Cryptocurrency involves risk of loss.*
