# Smart Wallet — product map

**Live product version:** **0.11.473** (extension `manifest.json`)  
**This repository:** documentation only — extension source is **not** published here.  
**Last aligned:** 2026-08-16

Smart Wallet and its LiFi backend are **separate folders** but parts of the **same product**. Keep backups and version histories separate. Treat changes to either folder as **cross-project** and compatibility-test quote / routes / step-transaction.

| Piece | Location |
|-------|----------|
| **Extension** | `C:\Users\levyr\Desktop\Gladiator-Wallet-0.6.55` |
| **Backend** | `C:\Users\levyr\Desktop\lifi backend fee distributor` |
| **Backend type** | Cloudflare Worker LiFi proxy: server-side `LIFI_API_KEY`, CORS allowlisting, endpoint rate limits, caching/deduplication, Durable Object shared quota |
| **Staging Worker** | `smart-wallet-lifi-proxy-staging` |
| **Staging URL** | `https://smart-wallet-lifi-proxy-staging.smart-wallet.workers.dev` |

The unpacked extension calls this backend for LiFi **quote / routes / step-transaction / status / tokens**. The Worker is the only place that talks to `li.quest` with the integrator key.

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
