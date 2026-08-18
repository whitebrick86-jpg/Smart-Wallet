# Smart Wallet — product map

**Live product version:** **0.11.298** (extension `manifest.json`)  
**This repository:** public extension source + documentation  
**Last aligned:** 2026-08-18  
**Batch 5:** Worker hardening, logo-host control, residual-fee removal, store package built  
**Worker:** **1.6.0** at `https://smart-wallet-lifi-proxy.smart-wallet.workers.dev`  
**Store ZIP:** built from this source as **0.11.298**; the zip artifact is not published here

Smart Wallet (this repository) and the separate LiFi Cloudflare Worker are parts of the **same product**. Worker source is **not** published here. Never copy a LiFi API key into the extension, this repository, source, logs, chat, or Git.

| Piece | Location |
|-------|----------|
| **Extension** | this repository (`manifest.json` at repo root) |
| **Production Worker** | `https://smart-wallet-lifi-proxy.smart-wallet.workers.dev` |
| **Worker version** | 1.6.0 |
| **Staging Worker** | explicit development option only; live MODE is production |

The unpacked extension calls the production Worker for LiFi **quote / routes / step-transaction / status / tokens**. The Worker is the only place that talks to `li.quest` with the integrator key.

**Never** copy the LiFi API key into the extension, this repository, source, logs, chat, or Git.

## Canonical fees (do not flatten)

| Path | Smart Wallet | LI.FI (quote-derived, currently) | Combined display |
|------|--------------|----------------------------------|------------------|
| Internal Jupiter swap | 45 bps (0.45%) | none | **0.45%** |
| Internal LiFi same-chain swap | 45 bps (0.45%) | 25 bps (0.25%), not Smart Wallet revenue | **0.70%** |
| Internal LiFi EVM-source bridge | 85 bps (0.85%) | 25 bps (0.25%), not Smart Wallet revenue | **1.10%** |
| Send / external DEX or bridge | 0 | n/a | **0%** from Smart Wallet |

The atomic verifier compares **only** the encoded Smart Wallet treasury distribution to exact 45/85 bps. Combined 70/110 bps is **display only** and must never be passed as the expected treasury fee. Direct 0x / Uniswap V2 / Pancake V3 fallbacks stay **disabled**. There is no separate fee transaction and no fee-free fallback.

Full fee copy: [FEE-DISCLOSURE.md](./Chrome-extension-store-for-reviewers/FEE-DISCLOSURE.md).
