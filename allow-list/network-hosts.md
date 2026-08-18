# Smart Wallet — host allowlist draft (replace global `https://*/*`)

**Purpose:** Build an explicit list of network hosts so `manifest.json` can drop  
`https://*/*` and `wss://*/*` without breaking stock product features.

**Source:** URLs found in `app.js`, `background.js`, `live-feeds.js`, `config.js`,  
`chain-registry.js`, `inject-allowlist.js`, and related extension files (inject snapshot **0.11.159**; live product **0.11.253**).

**Not the same as inject:**  
- **§A Network hosts** → `host_permissions` (extension fetch / WebSocket)  
- **§B Inject hosts** → `content_scripts.matches` (already allowlisted)  

---

## A. Network hosts (`host_permissions`)

Use these (or equivalent wildcards) for RPC, APIs, explorers, WC, logos.

### A1. Solana RPC / API

| Host | Role |
|------|------|
| `api.mainnet-beta.solana.com` | Default Solana RPC |
| `solana-rpc.publicnode.com` | Solana RPC failover |
| `solana.publicnode.com` | Solana RPC failover |
| `solana.drpc.org` | Solana RPC failover |
| `mainnet.helius-rpc.com` | Common Helius RPC host (custom RPC) |
| `api.helius.xyz` | Helius enhanced History / APIs |

**Optional later (user paste):** any other Helius/QuickNode/etc. host → `optional_host_permissions` when user saves custom RPC.

### A2. Ethereum mainnet RPC

| Host | Role |
|------|------|
| `ethereum.publicnode.com` | ETH RPC |
| `eth.llamarpc.com` | ETH RPC |
| `eth.drpc.org` | ETH RPC |
| `eth.meowrpc.com` | ETH RPC |
| `rpc.ankr.com` | Ankr multi-chain RPC path |
| `cloudflare-eth.com` | ETH RPC |
| `1rpc.io` | Public RPC aggregator (if still referenced) |

### A3. Base RPC

| Host | Role |
|------|------|
| `base.publicnode.com` | Base RPC |
| `mainnet.base.org` | Base official RPC |
| `base.llamarpc.com` | Base RPC |
| `base.drpc.org` | Base RPC |
| `base.meowrpc.com` | Base RPC |

### A4. Polygon RPC

| Host | Role |
|------|------|
| `polygon-bor.publicnode.com` | Polygon RPC |
| `polygon-rpc.com` | Polygon RPC |
| `polygon.llamarpc.com` | Polygon RPC |
| `polygon.drpc.org` | Polygon RPC |
| `gasstation.polygon.technology` | Gas station (if used) |

### A5. BNB Smart Chain RPC

| Host | Role |
|------|------|
| `bsc.publicnode.com` | BSC RPC |
| `bsc.drpc.org` | BSC RPC |
| `bsc-dataseed.binance.org` | BSC seed |
| `bsc-dataseed1.binance.org` | BSC seed |
| `bsc-dataseed2.binance.org` | BSC seed |
| `stream.binance.com` | Binance stream (if used for prices/feeds) |

### A6. Robinhood Chain RPC / explorer API

| Host | Role |
|------|------|
| `rpc.mainnet.chain.robinhood.com` | RH chain RPC |
| `4663.rpc.thirdweb.com` | RH / chain 4663 RPC |
| `robinhoodchain.blockscout.com` | RH explorer / API |

### A6b. Arbitrum One RPC / explorer (0.11.156+)

| Host | Role |
|------|------|
| `arb1.arbitrum.io` | Official Arbitrum One RPC |
| `arbitrum-one.publicnode.com` | Sequential failover |
| `arbitrum.drpc.org` | Sequential failover |
| `gateway.tenderly.co` | Tenderly public Arb (path `/public/arbitrum`) |
| `1rpc.io` | `1rpc.io/arb` |
| `arbiscan.io` | Explorer |
| `arbitrum.blockscout.com` | Blockscout catalog / icons |
| `arbitrum.io` | Portal (inject, not RPC) |

Removed after live probe: `arbitrum.llamarpc.com` (DNS fail).

### A6c. Optimism RPC / explorer (0.11.157+)

| Host | Role |
|------|------|
| `mainnet.optimism.io` | Official OP Mainnet RPC |
| `optimism.publicnode.com` | Sequential failover |
| `optimism.drpc.org` | Sequential failover |
| `1rpc.io` | `1rpc.io/op` |
| `optimistic.etherscan.io` | Explorer |
| `optimism.blockscout.com` | Blockscout |
| `optimism.io` | Portal (inject, not RPC) |

Removed after live probe: `optimism.meowrpc.com` (DNS fail).

### A6d. Avalanche C-Chain RPC / explorer (0.11.158+)

| Host | Role |
|------|------|
| `api.avax.network` | Official C-Chain RPC (`/ext/bc/C/rpc`) |
| `avalanche-c-chain-rpc.publicnode.com` | Sequential failover |
| `avalanche.drpc.org` | Sequential failover |
| `1rpc.io` | `1rpc.io/avax/c` |
| `snowtrace.io` | C-Chain explorer |
| `avax.network` | Portal (inject, not RPC) |
| `stream.binance.com` | `AVAXUSDT` ticker when Avalanche is the active net |

Removed after live probe: `rpc.ankr.com/avalanche` (no `chainId`), `avalanche.public-rpc.com` (403).

### A7. Bitcoin / Sui

| Host | Role |
|------|------|
| `blockstream.info` | Bitcoin API |
| `mempool.space` | Bitcoin API / broadcast |
| `sui-mainnet-endpoint.blockvision.org` | Sui RPC |
| `rpc-mainnet.suiscan.xyz` | Sui RPC |
| `suiscan.xyz` | Sui explorer |

### A8. Internal swap / bridge / prices

| Host | Role |
|------|------|
| `lite-api.jup.ag` | Jupiter swap + prices |
| `smart-wallet-lifi-proxy-staging.smart-wallet.workers.dev` | Unpacked extension LiFi quote / routes / step-transaction (staging Worker) |
| `li.quest` | LiFi upstream used **by the Worker only**. The extension must not send the integrator key here. |
| `scan.li.fi` | LiFi status / scan |
| `li.fi` | LiFi related (if called) |
| `api.coingecko.com` | Fiat prices |
| `assets.coingecko.com` | Token logo CDN |
| `coin-images.coingecko.com` | Token logo CDN |
| `api.dexscreener.com` | Market / token search (if used in UI) |
| `cdn.dexscreener.com` | Dexscreener images (if used) |
| `api.geckoterminal.com` | Token meta (if used) |

### A9. Explorers (history / links / APIs)

| Host | Role |
|------|------|
| `solscan.io` | Solana explorer |
| `etherscan.io` | ETH explorer |
| `api.etherscan.io` | ETH explorer API |
| `basescan.org` | Base explorer |
| `api.basescan.org` | Base explorer API |
| `polygonscan.com` | Polygon explorer |
| `api.polygonscan.com` | Polygon explorer API |
| `bscscan.com` | BSC explorer |
| `api.bscscan.com` | BSC explorer API |
| `arbiscan.io` | Arbitrum explorer |
| `api.arbiscan.io` | Arbitrum API |
| `optimistic.etherscan.io` | Optimism explorer |
| `api-optimistic.etherscan.io` | Optimism API |
| `snowtrace.io` | Avalanche C-Chain explorer |
| `eth.blockscout.com` | Blockscout ETH |
| `base.blockscout.com` | Blockscout Base |
| `polygon.blockscout.com` | Blockscout Polygon |
| `bsc.blockscout.com` | Blockscout BSC |
| `arbitrum.blockscout.com` | Blockscout Arbitrum |
| `optimism.blockscout.com` | Blockscout Optimism |

### A10. Token lists / logos / metadata CDNs

| Host | Role |
|------|------|
| `tokens.uniswap.org` | Uniswap token list |
| `tokens.pancakeswap.finance` | PCS token list |
| `tokens.1inch.io` | 1inch token logos |
| `raw.githubusercontent.com` | Token list / logo raw files |
| `arweave.net` | Token logos (e.g. curated mints) |
| `ipfs.io` | IPFS gateway logos |
| `gateway.ipfs.io` | IPFS gateway |
| `storage.googleapis.com` | e.g. JitoSOL metadata images |
| `xstocks-metadata.backed.fi` | xStock logos |
| `icons.duckduckgo.com` | Favicons for dApp origins |
| `www.google.com` | Favicon fallback (if still used) |

### A11. WalletConnect / Reown (if WC used)

| Host | Role |
|------|------|
| `relay.walletconnect.org` | WC relay |
| `relay.walletconnect.com` | WC relay (alt) |
| `rpc.walletconnect.org` | WC RPC helper |
| `verify.walletconnect.org` | WC verify |
| `verify.walletconnect.com` | WC verify alt |
| `pulse.walletconnect.org` | WC analytics/pulse (if SDK hits it) |
| `echo.walletconnect.com` | WC echo |
| `api.pay.walletconnect.com` | WC pay API (if SDK hits it) |
| `dashboard.walletconnect.com` | Docs/links only; optional |

**WSS:** for each WC host that uses sockets, also grant `wss://relay.walletconnect.org/*` (and same for other WSS hosts actually used).

### A12. Ledger live / device services (if extension calls them)

| Host | Role |
|------|------|
| `cdn.live.ledger.com` | Ledger assets (if used) |
| `crypto-assets-service.api.ledger.com` | Ledger assets API (if used) |
| `explorers.api.live.ledger.com` | Ledger explorers API (if used) |
| `nft.api.live.ledger.com` | Ledger NFT API (if used) |

*(Only include if runtime traffic actually hits them; some appear from docs/bundles.)*

### A13. Local dev

| Pattern | Role |
|---------|------|
| *(localhost / 127.0.0.1)* | **Not** in the production package |

---

## Suggested `host_permissions` shape (draft)

Exact permission strings (HTTPS). Prefer apex wildcards only where the product already uses many subdomains:

```text
# Solana
https://api.mainnet-beta.solana.com/*
https://solana-rpc.publicnode.com/*
https://solana.publicnode.com/*
https://solana.drpc.org/*
https://mainnet.helius-rpc.com/*
https://api.helius.xyz/*

# Ethereum
https://ethereum.publicnode.com/*
https://eth.llamarpc.com/*
https://eth.drpc.org/*
https://eth.meowrpc.com/*
https://rpc.ankr.com/*
https://cloudflare-eth.com/*
https://1rpc.io/*

# Base
https://base.publicnode.com/*
https://mainnet.base.org/*
https://base.llamarpc.com/*
https://base.drpc.org/*
https://base.meowrpc.com/*

# Polygon
https://polygon-bor.publicnode.com/*
https://polygon-rpc.com/*
https://polygon.llamarpc.com/*
https://polygon.drpc.org/*
https://gasstation.polygon.technology/*

# BSC
https://bsc.publicnode.com/*
https://bsc.drpc.org/*
https://bsc-dataseed.binance.org/*
https://bsc-dataseed1.binance.org/*
https://bsc-dataseed2.binance.org/*
https://stream.binance.com/*

# Robinhood chain
https://rpc.mainnet.chain.robinhood.com/*
https://4663.rpc.thirdweb.com/*
https://robinhoodchain.blockscout.com/*

# Bitcoin / Sui
https://blockstream.info/*
https://mempool.space/*
https://sui-mainnet-endpoint.blockvision.org/*
https://rpc-mainnet.suiscan.xyz/*
https://suiscan.xyz/*

# Swap / bridge / prices
https://lite-api.jup.ag/*
https://li.quest/*
https://scan.li.fi/*
https://li.fi/*
https://api.coingecko.com/*
https://assets.coingecko.com/*
https://coin-images.coingecko.com/*
https://api.dexscreener.com/*
https://cdn.dexscreener.com/*
https://api.geckoterminal.com/*

# Explorers
https://solscan.io/*
https://etherscan.io/*
https://api.etherscan.io/*
https://basescan.org/*
https://api.basescan.org/*
https://polygonscan.com/*
https://api.polygonscan.com/*
https://bscscan.com/*
https://api.bscscan.com/*
https://arbiscan.io/*
https://api.arbiscan.io/*
https://eth.blockscout.com/*
https://base.blockscout.com/*
https://polygon.blockscout.com/*
https://bsc.blockscout.com/*

# Logos / token lists
https://tokens.uniswap.org/*
https://tokens.pancakeswap.finance/*
https://tokens.1inch.io/*
https://raw.githubusercontent.com/*
https://arweave.net/*
https://ipfs.io/*
https://gateway.ipfs.io/*
https://storage.googleapis.com/*
https://xstocks-metadata.backed.fi/*
https://icons.duckduckgo.com/*

# WalletConnect
https://relay.walletconnect.org/*
https://relay.walletconnect.com/*
https://rpc.walletconnect.org/*
https://verify.walletconnect.org/*
https://verify.walletconnect.com/*
https://pulse.walletconnect.org/*
https://echo.walletconnect.com/*
wss://relay.walletconnect.org/*
wss://relay.walletconnect.com/*

# Localhost is not in the production package.
```

**Custom Solana RPC:** keep as optional permission flow later, e.g. request access when user saves a URL whose host is not already listed.

---

## B. Inject / content-script hosts (already separate)

These are for **page inject** only (not a substitute for §A). Full list lives in `inject-allowlist.js`. Apex examples:

**Solana trading:** `jup.ag`, `jupiter.ag`, `pump.fun`, `raydium.io`, `orca.so`, `tensor.trade`, `drift.trade`, `mango.markets`, `kamino.finance`, `sanctum.so`, `sol-incinerator.com`, `meteora.ag`, `phoenix.trade`, `birdeye.so`, `dexscreener.com`, `photon-sol.tinyastro.io`, `bullx.io`, `trojan.com`, `fluxbot.xyz`, `bonkbot.io`, `jito.network`, …

**EVM DEX / DeFi / bridges:** `uniswap.org`, `pancakeswap.finance`, `sushi.com`, `1inch.io`, `curve.fi`, `relay.link`, `li.fi`, `jumper.exchange`, `stargate.finance`, `base.org`, `etherscan.io`, `aave.com`, …

Do **not** merge §B into network permissions blindly: inject list is large and is about **pages**, not RPCs.

---

## C. Noise / do not need in host_permissions

Found in tree but typically **docs / comments / libraries**, not product network:

- `developer.mozilla.org`, `docs.ethers.org`, `crypto.stackexchange.com`, `eprint.iacr.org`, `en.bitcoin.it`, `ens.domains`, `feross.org`, `oxlib.sh`, `www.w3.org`, `www.jsdelivr.com`, `github.com` (unless you fetch release assets), `t.me`, `smart-wallet.wallet` (fake/internal), `gladiator.wallet`, `sola.na`, `robinhood.com` (marketing), testnet explorer APIs if mainnet-only product.

---

## D. Count summary

| Category | Approx. hosts |
|----------|----------------|
| Core RPCs (all chains) | ~35–40 |
| Swap / bridge / prices | ~10 |
| Explorers | ~15 |
| Logos / lists | ~12 |
| WalletConnect | ~8 + WSS |
| Local | 4 patterns |
| **Network allowlist total (stock)** | **~80–90 permission lines** |
| Inject allowlist (dApps) | **~80+ apex hosts** (separate) |

---

## E. Next steps (when you want implementation)

1. Freeze §A list (trim any unused after a network traffic pass).  
2. Replace `https://*/*` / `wss://*/*` in `manifest.json` with §A.  
3. Keep `content_scripts.matches` / inject-allowlist as §B.  
4. Add optional permission path for user custom RPC.  
5. Smoke-test: Home balances, Send, History, Swap, Bridge, Jupiter dApp, Uniswap, Ledger connect, WC.

---

*Draft only — not applied to `manifest.json` yet. Built from static URL scan of the 0.11.0 tree.*
