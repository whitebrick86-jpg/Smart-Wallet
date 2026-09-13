# Smart Wallet — product map

**Live product version:** **0.11.698** (extension `manifest.json`)  
**This repository:** documentation only — extension source is **not** published here.  
**Last aligned:** 2026-09-13

Smart Wallet and its LiFi backend are **separate folders** but parts of the **same product**. Keep backups and version histories separate. Treat changes to either folder as **cross-project** and compatibility-test quote / routes / step-transaction.

| Piece | Location |
|-------|----------|
| **Extension (production)** | `C:\Users\levyr\Desktop\Smart-Wallet-0.6.55` |
| **Extension (React UI experiment)** | `C:\Users\levyr\Desktop\React-Wallet` (Smart Wallet R) |
| **Backend** | `C:\Users\levyr\Desktop\lifi backend fee distributor` |
| **Backend type** | Cloudflare Worker LiFi proxy: server-side `LIFI_API_KEY`, CORS allowlisting, endpoint rate limits, caching/deduplication, Durable Object shared quota |
| **Live LiFi MODE** | **production** |
| **Production Worker** | `smart-wallet-lifi-proxy` · service version `1.6.4` |
| **Production URL** | `https://smart-wallet-lifi-proxy.smart-wallet.workers.dev` |
| **Staging Worker** | `smart-wallet-lifi-proxy-staging` (explicit developer option; not the live MODE) |
| **Staging URL** | `https://smart-wallet-lifi-proxy-staging.smart-wallet.workers.dev` |

The live product path calls the **production** Worker for LiFi **quote / routes / step-transaction / status / tokens**. Staging remains an explicit developer option. The Worker is the only place that talks to `li.quest` with the integrator key.

**Never** copy the LiFi API key into the extension, this repository, source, logs, chat, or Git.

## Canonical fees (do not flatten)

| Path | Smart Wallet | LI.FI (quote-derived, currently) | Combined display |
|------|--------------|----------------------------------|------------------|
| Internal Jupiter swap | 45 bps (0.45%) | none | **0.45%** |
| Internal LiFi same-chain swap | 45 bps (0.45%) | 25 bps (0.25%), not Smart Wallet revenue | **0.70%** |
| Internal LiFi EVM-source bridge | 85 bps (0.85%) | 25 bps (0.25%), not Smart Wallet revenue | **1.10%** |
| Send / external DEX or bridge | 0 | n/a | **0%** from Smart Wallet |

The atomic verifier compares **only** the encoded Smart Wallet treasury distribution to exact 45/85 bps. Combined 70/110 bps is **display only** and must never be passed as the expected treasury fee. Direct 0x / Uniswap V2 / Pancake V3 fallbacks stay **disabled**. There is no separate fee transaction.

**Best-effort platform fees.** Never block an otherwise-safe swap or bridge solely because a platform-fee insert failed. Never sign malformed, misdirected, or unverifiable fee payloads.

- **Jupiter:** if a fee-bearing quote cannot be used, the client may rebuild a fresh fee-free quote and sign **only** that rebuild.
- **LiFi routes:** narrow fee-unavailable carve-out; no client fee-free rebuild yet; never sign a flagged fee payload.

Full fee copy: [FEE-DISCLOSURE.md](./Chrome-extension-store-for-reviewers/FEE-DISCLOSURE.md).
