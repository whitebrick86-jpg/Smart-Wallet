# Smart Wallet — Why global host permissions

**Product:** Smart Wallet (Chrome / Opera Manifest V3 extension)  
**Audience:** Users, operators, security reviewers, Chrome Web Store reviewers  
**Related code concepts:** `host_permissions` vs content-script `matches` / inject allowlist  
**Docs snapshot:** 0.11.298

Unpacked production uses one exact Worker host:

```text
https://smart-wallet-lifi-proxy.smart-wallet.workers.dev/*
```

---

## 1. Short answer

Required `host_permissions` are **named** RPC, API, explorer, and WalletConnect hosts.

`https://*/*` and `wss://*/*` remain **optional only**. They exist so a user-pasted Advanced / custom RPC origin (for example a saved QuickNode Solana URL) can be requested at runtime via `chrome.permissions.request` for **that exact origin**. The extension does not grant `https://*/*` at install.

**That is not “inject the wallet into every website.”**

**Page inject** (provider on dApps) is a **separate, allowlisted** mechanism. Only listed DEX/DeFi hosts get `content_scripts` / provider install. Localhost and `127.0.0.1` are **not** in the production/store inject list or required host permissions.

| Permission class | Scope in Smart Wallet | Purpose |
|------------------|----------------------|---------|
| **Required `host_permissions`** | Named RPC / API / explorer / WC hosts | Stock network access |
| **Optional `https://*/*` + `wss://*/*`** | User-selected origin at runtime | Advanced / custom RPC only |
| **Content scripts / inject allowlist** | Listed dApp hosts | Wallet Standard + EIP-1193 |
| **Localhost** | Development builds only | Never shipped in production |

This split is intentional architecture, not an oversight.

---

## 2. What “global host permission” means in MV3

In Chrome extensions:

- **`host_permissions`** grant the extension the right to make **cross-origin network requests** to matching hosts (and, with scripting, to interact with matching tabs when other APIs require it).
- **`content_scripts.matches`** control **which pages** automatically receive injected scripts.

A wallet that only listed a fixed set of RPC domains would **fail** whenever:

- a public RPC host rotates or dies,
- the user adds a private Helius / QuickNode / Ankr / self-hosted URL,
- a swap/bridge aggregator returns a transaction that must be broadcast to a **different** HTTPS endpoint than the default list,
- WalletConnect / Reown uses relay hosts not frozen at ship time,
- a new chain or explorer is added without a full extension re-release.

Multi-chain non-custodial wallets routinely need **network flexibility**. Smart Wallet keeps **inject narrow** and **network wide** so those two risks stay separate.

---

## 3. Why *this* architecture requires it

### 3.1 Multi-chain, multi-RPC consensus (not one fixed hostname)

Each chain maintains an **ordered list of HTTPS RPC URLs** (and often WSS for activity). The wallet:

- tries several public endpoints,
- fails over on 429 / timeout / bad node,
- for some EVM balance paths uses **multi-RPC consensus** so one flaky node returning `0x0` does not brick UX.

Those hosts change over time (publicnode, llamarpc, ankr, chain official RPCs, Robinhood chain, Base, BSC seeds, Solana public + custom). **Encoding every possible RPC origin into a frozen allowlist** is brittle and user-hostile.

**Without broad HTTPS access:** balances, send, history, and dApp-backed reads break whenever the next RPC hostname is not pre-declared.

### 3.2 User-configured custom Solana RPC (Helius and others)

Settings allow a **user-pasted** Solana RPC URL (often Helius with an `api-key` query param). That origin is **chosen by the user at runtime**, not known at store submit time.

Architecture rule:

- Custom RPC is used for Solana reads / History enhancement.
- The API key stays **local** in extension storage.
- The extension must be allowed to `fetch` that **user-chosen origin**.

**Without `https://*/*` (or a perfect optional-permission UX for every paste):** custom RPC and Helius History either cannot work or require a permission prompt on every new host — which is easy to get wrong in MV3 and still fails for opaque RPC domains behind CDNs.

### 3.3 Internal Swap / Bridge aggregators

In-wallet Swap and Bridge call **third-party HTTPS APIs** that are not “the chain RPC”:

| Feature | Typical network targets |
|---------|-------------------------|
| Solana swap | Jupiter (and related) HTTPS APIs + Solana RPC broadcast |
| EVM swap | LiFi-style quote + route build + chain RPC send |
| Bridge | Cross-chain quote/status APIs + origin/destination RPCs |
| Platform fee settle | Extra Jupiter/LiFi convert then transfer to treasury |

Quote services, status endpoints, and token metadata hosts are **not stable single domains forever**. Routes and partners change. The wallet must be able to call **whatever HTTPS URL the route needs** to build and land the trade the user approved in-wallet.

**Without global HTTPS:** internal DEX can silently lose routes or fail mid-flight when an aggregator or partner host is outside a static list.

### 3.4 WalletConnect / Reown relays

WalletConnect needs the **full wallet window** and HTTPS/WSS to **relay infrastructure** (project-specific / vendor hosts). Relay endpoints are not identical to “only jup.ag” or “only one RPC.”

**Without broad network access:** pairing and session keep-alive fail or require constant allowlist chases after vendor infra changes.

### 3.5 Free live data (idle-first), not paid market streams

`live-feeds.js` opens:

- **one** Binance market WebSocket for the **active** native ticker (`wss://…`),
- Solana **logsSubscribe** over **WSS** (custom or public Solana WS URLs).

EVM `newHeads` is **not** used (idle-load control). HTTP price fallbacks still hit public HTTPS price APIs (e.g. CoinGecko, Jupiter price batches).

**`wss://*/*`** exists so Solana activity and market streams can use the same multi-host / custom-RPC story as HTTPS, without freezing every WS vendor in the manifest.

### 3.6 Explorers, metadata, and secondary HTTPS

Secondary calls (explorer links are user-navigation; some metadata / logo / discovery paths may still `fetch`) also benefit from not hard-coding every CDN. The core justification remains **RPC + swap/bridge + WC + prices**, not “read the whole web for fun.”

### 3.7 Localhost

`http://localhost/*` and `http://127.0.0.1/*` support local dApp development and local test pages. They are **not** a substitute for global HTTPS; they are a small, expected developer/user convenience.

---

## 4. What global host permission is *not* used for

### 4.1 Not universal dApp injection

Provider install is gated by **`inject-allowlist.js`** and matching **`content_scripts`** / web-accessible resource lists (Jupiter, Raydium, Uniswap-class hosts, bridges, etc.).

- Random sites do **not** get `window.solana` / EIP-1193 from Smart Wallet by default.
- Approvals stay in the **wallet UI**, not a fake in-page modal controlled by the dApp.

So: **broad network ≠ broad inject.**

### 4.2 Not Smart Wallet cloud custody

There is **no** Smart Wallet server that receives seeds or vault passwords. Network calls go from the user’s browser to **public or user-chosen** infrastructure. Global host permission does not create a Smart Wallet backend; it allows the client to reach decentralized and third-party endpoints.

### 4.3 Not unrestricted reading of every tab’s DOM

Host permission alone does not mean the extension scrapes every page. Content-script execution remains allowlisted. Service worker / popup fetches are for wallet operations the user initiated (open wallet, swap, sync, connect, etc.).

---

## 5. Security model that goes with this choice

Broad network access increases **what a compromised extension could reach**. Smart Wallet offsets that with product controls:

| Control | Role |
|---------|------|
| **Non-custodial keys** | Seed / keys local or on Ledger; no cloud seed store |
| **Vault** | AES-256-GCM + high-cost PBKDF2 when password is on |
| **Auto-lock** | UI + service-worker alarms; secrets cleared on lock |
| **Allowlisted inject** | Limits phishing surface of “wallet appears on every site” |
| **User approve for dApp trust** | First connect is explicit |
| **`eth_sign` disabled** | Prefer `personal_sign` / typed data |
| **Idle-first networking** | Avoids permanent spam to all hosts “just because we can” |
| **CSP** | Extension pages: `script-src 'self'` (no remote extension JS) |

Reviewers should evaluate **host_permissions** together with **inject allowlist + vault + no remote code**, not in isolation.

---

## 6. Alternatives we considered (and why they are not the default)

### 6.1 Static allowlist of every known RPC + API host

**Pros:** Smaller declared surface; easier first-pass store optics.  
**Cons:** Breaks custom RPC; breaks when public RPCs change hostnames; enormous maintenance; still incomplete for WC/aggregators; we already saw **product breakage** when hosts were narrowed aggressively without optional-permission UX.

### 6.2 Optional host permissions requested at runtime

**Pros:** Minimum permanent grant; user sees prompts for new origins.  
**Cons:** MV3 prompts are easy to get wrong (must be user-gesture-tied); multi-RPC failover would spam prompts or fail closed; WalletConnect and aggregator fan-out become fragile; poor UX for a multi-chain wallet.

A hybrid (“core list + optional custom RPC”) is a possible future engineering project. It is **not** free and must not regress send/swap/dApp reliability.

### 6.3 Proxy all traffic through a Smart Wallet backend

**Pros:** Extension only needs one host.  
**Cons:** Centralizes privacy and availability; conflicts with non-custodial “talk to the chain yourself” design; creates a server we deliberately do not run for core use.

---

## 7. Justification text suitable for store / review notes

Use language consistent with the product:

> Smart Wallet is a non-custodial multi-chain crypto wallet.  
> **`https://*/*` and `wss://*/*`** allow the extension to communicate with blockchain RPC endpoints, swap/bridge aggregators, price feeds, WalletConnect relays, and **user-configured custom RPC URLs** that cannot be fully known at publish time.  
> Multi-RPC failover and custom Solana RPC (e.g. Helius) are core reliability features.  
> **Provider injection is not global:** content scripts run only on an allowlisted set of DeFi/dApp hosts.  
> The extension does not operate a custodial backend; keys remain on device or on Ledger.  
> Localhost permissions support local development and local dApps only.

---

## 8. User-facing plain language

**Why does Smart Wallet ask for access to all sites’ network?**

Think of it like a browser that must call many different “banks” (blockchains) and “exchanges” (DEXes) whose addresses change, plus **your** private RPC if you add one.

- It does **not** mean every website gets a Smart Wallet popup injected.  
- It means when **you** open the wallet to check balances, swap, bridge, or connect a listed dApp, the wallet can reach the HTTPS/WSS services required to do that job.  
- dApp connection on the open web is limited to **supported / allowlisted** sites.

---

## 9. Operator checklist (keep the story true)

When changing architecture, keep this document honest:

- [ ] Inject remains **allowlist-only** (do not expand inject to `https://*/*`)
- [ ] No remote-loaded extension logic (CSP stays `'self'`)
- [ ] Custom RPC still documented in Settings / docs
- [ ] Privacy policy still lists third-party classes (RPCs, Jupiter, LiFi, prices, WC, optional Helius)
- [ ] Store listing permission justification matches this file
- [ ] If hosts are ever narrowed, ship optional permission + regression tests for custom RPC and multi-RPC failover first

---

## 10. Related documents

| Document | Relevance |
|----------|-----------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Surfaces, inject vs SW, live feeds, fees |
| [LOADS.md](./LOADS.md) | What we ping and when (idle-first) |
| [PRIVACY-POLICY.md](./PRIVACY-POLICY.md) | Third parties / no seed server |
| [STORE-LISTING.txt](./STORE-LISTING.txt) | Public store copy |
| [CHROME-WEB-STORE-READINESS.md](./CHROME-WEB-STORE-READINESS.md) | Store gap analysis (includes host risk) |
| [DOCUMENTATION.txt](./DOCUMENTATION.txt) | End-user RPC / dApp sections |

---

## 11. Bottom line

| Question | Answer |
|----------|--------|
| Why global HTTPS/WSS? | Multi-chain multi-RPC, **user custom RPC**, aggregators, WC, live WSS — runtime hosts cannot be fully frozen |
| Does that inject everywhere? | **No** — inject is allowlisted |
| Is it required by *this* architecture? | **Yes** for reliable non-custodial multi-chain operation without a Smart Wallet proxy server |
| Safer future option? | Optional permissions / hybrid lists — only after UX + failover are proven |

**Global host permission is the network capability layer. Allowlisted inject is the page-trust layer. Smart Wallet needs both layers kept distinct — and the network layer kept flexible — to match how multi-chain wallets actually work.**

---

*Not financial advice. Cryptocurrency involves risk of loss.*
