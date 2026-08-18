# Smart Wallet — Architecture

**Product:** Smart Wallet (Chrome / Opera MV3 extension)  
**Live product:** **0.11.257** · **Architecture snapshot:** **0.11.159** (shared EVM path, sequential RPC, 45/85 bps)  
**Repository:** Documentation only — extension source is **not** published here.  
**Product map:** [PRODUCT.md](./PRODUCT.md) (extension + LiFi Worker are separate folders of the same product)  
**Last architecture update:** 2026-08-16 (live version / Worker transport noted; 0.11.159 body kept as the shipped-module snapshot)  

This document describes **what is implemented** in the Load-unpacked product build.  
Items are marked **Shipped**, **Partial**, or **Planned**. Nothing is labeled shipped unless present in the extension tree and covered by `tools/verify-extension.ps1` syntax/version gates (and unit smoke where noted).

---

## 1. Product capabilities

| Capability | Notes |
|------------|--------|
| Chains (software) | Solana, Ethereum, Bitcoin, Polygon, Sui, BNB Smart Chain, Robinhood ETH (4663), Base, **Arbitrum One (42161)**, **Optimism (10)**, **Avalanche C-Chain (43114)** |
| Ledger | Solana + EVM after Link EVM (not BTC / Sui on device) |
| Keys | Local software vault (always encrypted at rest) or hardware Ledger |
| dApps | Wallet Standard + EIP-1193 on an **allowlisted** host set only |
| In-wallet | Send, Receive (Onramper buy), Swap, Bridge, History, PnL |
| Live data | Idle-first free market WS + Solana mentions; HTTP fallbacks |
| Platform fees | **0.45%** Smart Wallet swap · **0.85%** Smart Wallet bridge · LiFi currently adds **0.25%** on EVM routes (combined **0.70%** / **1.10%**) · none on Send / external DEX |

**No Smart Wallet cloud custody.** Network calls go from the user’s browser to public RPCs / price APIs / optional user Helius / WalletConnect / Onramper. Unpacked LiFi quote/routes/step go to the **staging Cloudflare Worker**, not to `li.quest` with a key in the extension.

---

## 2. High-level runtime

```text
+------------------------------------------------------------------+
|                     USER BROWSER PROFILE                         |
|                                                                  |
|  +---------------------------+    chrome.runtime   +-----------+ |
|  | Wallet UI                 | <-----------------> | Service   | |
|  | popup.html / index.html   |     messages        | worker    | |
|  | (script stack below)      |                     | background| |
|  +------------+--------------+                     +-----+-----+ |
|               | storage                                  |       |
|               v                                          |       |
|  +---------------------------+                           |       |
|  | chrome.storage.local      | <-------------------------+       |
|  | + localStorage mirror     |  smart_wallet_v1 (+ legacy)       |
|  +---------------------------+                                   |
|                                                                  |
|  +----------------+  postMessage  +---------------------------+  |
|  | content-script | <-----------> | MAIN: page-boot, injected |  |
|  | + allowlist    |               | page-evm-lite             |  |
|  +-------+--------+               +-------------+-------------+  |
|          | allowlisted hosts only               |                |
+----------|--------------------------------------|----------------+
           v
   Public HTTPS/WSS via RPC Gateway: chain RPCs, Jupiter, CoinGecko,
   Binance ticker, optional Helius, WalletConnect/Reown, explorers
```

### 2.1 UI script load order (critical)

Order in `popup.html` / `index.html` (version-query cache bust):

1. `config.js`
2. **Network foundation:** `chain-registry.js` → `cache-coordinator.js` → `rpc-gateway.js` → `rpc-manager.js`
3. **Coordination:** `sw-events.js` → `state-coordinator.js`
4. **Transaction:** `tx-intent.js` → `transaction-manager.js`
5. **Product managers:** `portfolio-manager.js` → `price-manager.js` → `history-manager.js` → `swap-manager.js` → `bridge-manager.js`
6. **Helpers:** `live-feeds.js`, `fee-helpers.js`, `address-guards.js`, `clipboard-guard.js`, `history-filter.js`, `onramp.js`
7. **UI logo:** `ui/ui-logo.js` (product mark; must load **before** `app.js`)
8. **`app.js`** (product UI + majority of business logic)
9. **`manager-bootstrap.js`** (registers adapters; event wiring)
10. **`ui/ui-shell.js`**, `ui-failsafe.js`

Service worker loads: `inject-allowlist.js`, then `chain-registry.js`, `cache-coordinator.js`, `rpc-gateway.js` via `importScripts`.  
Content scripts: `inject-allowlist.js` → **`ui/ui-logo.js`** → `content-script.js`.

### 2.2 Three systems (separation of concerns)

```text
NETWORK                         WALLET                          dAPP
-------                         ------                          ----
Chain Registry                  Vault (app.js)                  Allowlist
RPC Gateway                     Accounts                        Injected provider
RPC Manager (UI)                Signer (SW / offscreen)         Content bridge
Cache Coordinator               Ledger (device keys)            Service worker
Provider score / cooldown       Transaction Manager             User approval UI
                                Events / State helpers          (never fake modal)
```

- Networking modules never hold private keys.  
- dApp inject never bypasses in-wallet approval.  
- Signing never runs inside the RPC gateway.

---

## 3. RPC architecture (foundation — do not redesign)

**Shipped since 0.11.33; intelligence extended in 0.11.34.**

| Module | Global / API | Role |
|--------|--------------|------|
| `chain-registry.js` | `SmartWalletChainRegistry` | Single `CHAINS` + `EVM_NETWORKS` for UI **and** SW |
| `cache-coordinator.js` | `SmartWalletCache` | TTL get/set + in-flight promise dedupe |
| `rpc-gateway.js` | `SmartWalletRpcGateway` | Sequential multi-RPC execute, timeouts, cooldown |
| `rpc-manager.js` | `SmartWalletRpc` | UI: prefer SW proxy, then direct gateway |

### 3.1 Rules (enforced)

1. **One host list** — UI and SW share registry (no drift).  
2. **Sequential failover** — never parallel fan-out of free RPCs.  
3. **One request → one healthy provider** — try best host; fall through only on failure / 429 / timeout.  
4. **Hard timeouts** — especially `eth_getLogs` (prevents stuck Sync).  
5. **No keys in gateway** — public JSON-RPC only.  
6. **Helius** — optional user key for Solana **History**, not continuous balance spam.

### 3.2 Provider intelligence (0.11.34 — Shipped)

| Feature | Behavior |
|---------|----------|
| Host stats | ok / fail / rateLimit / latency samples |
| Cooldown | Soft ban after 429 (~45s) |
| **Score** | reliability + inverse latency + recency − 429 penalty − optional cost weight |
| `orderHosts` | Ranks by score (tie-break original order) |
| Observability | `getHealthSnapshot()`; last provider per purpose |
| Custom RPC | Optional host permission when user pastes URL |

Still **sequential** — scoring only chooses order, not parallel blast.

### 3.3 SW message types (RPC)

| Message | Who | Purpose |
|---------|-----|---------|
| `smart-wallet-sol-rpc` | Trusted wallet UI | Solana JSON-RPC via gateway |
| `smart-wallet-evm-rpc` | Trusted wallet UI | EVM JSON-RPC via gateway |
| `smart-wallet-rpc-stats` | Trusted wallet UI | Health snapshot |

---

## 4. Transaction pipeline (0.11.34)

### 4.1 Target lifecycle (implemented as manager)

```text
UI / dApp
    |
    v
Transaction Intent          (tx-intent.js — normalized, no secrets)
    |
    v
Validation                  (local intent checks; path-specific sim in app.js)
    |
    v
User Approval               (wallet UI only)
    |
    v
Signer                      software | ledger | offscreen
    |
    v
Transaction Manager         states below
    |
    v
RPC Gateway                 sequential broadcast / confirm votes
    |
    v
Broadcast → multi-RPC Confirmation / Reconciliation
    |
    v
Events (tx:*, wallet:activity) → cache / light portfolio refresh → UI
```

### 4.2 States (`SmartWalletTx.States`)

```text
CREATED → SIMULATING → AWAITING_APPROVAL → SIGNED → BROADCAST
  → CONFIRMING → RECONCILING → CONFIRMED → FINALIZED
  | FAILED   (only with reliable on-chain evidence, e.g. receipt status=0)
```

**Critical rules:**

1. **(0.11.35)** Confirmation lookup "not found" / empty receipt → **CONFIRMING** / **RECONCILING**, never automatic **FAILED**, when a broadcast hash/signature already exists.  
2. **(0.11.36)** UI must validate tx identity **per chain** (`isValidSendTxIdentity`). Requiring EVM `0x`+64-hex for Solana/BTC/Sui caused false **"no valid transaction hash"** after successful broadcast.

### 4.3 Broadcast vs confirmation (P0 fix)

| Stage | Meaning |
|-------|---------|
| Broadcast succeeded | Network returned a valid `0x…` hash (or Solana signature) |
| Confirming | Hash stored; sequential multi-RPC reconciliation in progress |
| Pending after timeout | Still **not** failed — UI: "Submitted — confirming…" |
| Failed | Receipt `status=0` / proven on-chain revert, or **no hash** from broadcast response |

Send/swap software + Ledger paths call `finalizeEvmAfterBroadcast(hash, …)` after broadcast:

1. `SmartWalletTx.recordBroadcast` — persist identity (`smart_wallet_tx_lifecycle_v1`)
2. Multi-RPC `waitEvmConfirmed` (sequential via gateway)
3. Success → CONFIRMED/FINALIZED
4. On-chain revert → FAILED
5. Still pending → return hash; toast confirming; light balance refresh

**Do not** rebroadcast the same signed payload merely because lookup missed.

### 4.4 Status table

| Piece | Status |
|-------|--------|
| Intent model `SmartWalletTxIntent` | **Shipped** |
| Lifecycle + CONFIRMING/RECONCILING | **Shipped** (0.11.35) |
| Durable tx identity + `resumePending` | **Shipped** |
| `finalizeEvmAfterBroadcast` on Send / Ledger / Swap EVM | **Shipped** |
| False "not found after broadcast" (EVM confirm lag) | **Fixed** (0.11.35) |
| False "no valid transaction hash" (UI forced EVM format) | **Fixed** (0.11.36) |
| Regression tests `tools/test-tx-lifecycle.js` | **Shipped** (22 cases) |
| All paths only via `runLifecycle` | **Partial** — post-broadcast finalize shared; full builder migrate ongoing |
| BTC/Sui full adapters | **Partial** |

**Security:** managers never receive seed/private keys.

---

## 5. Product managers (0.11.34)

| Manager | Global | Status | Notes |
|---------|--------|--------|-------|
| Portfolio | `SmartWalletPortfolio` | **Shipped (orchestration)** | Stages: cache → native → known tokens → activity → discovery → prices; adapters in bootstrap |
| Price | `SmartWalletPrices` | **Shipped (orchestration)** | Dedupe/cache; adapters call existing CoinGecko / Jupiter paths |
| History | `SmartWalletHistory` | **Shipped (orchestration)** | Unified API + durable local cache; Sol/EVM/BTC/Sui/Helius via adapters |
| Swap | `SmartWalletSwap` | **Shipped (façade)** | Fee **45 bps (0.45%)** via `FeeHelpers` |
| Bridge | `SmartWalletBridge` | **Shipped (façade)** | Fee **85 bps (0.85%)** via `FeeHelpers` |
| Events | `SmartWalletEvents` | **Shipped** | `tx:*`, `portfolio:*`, `price:*`, `history:*`, `wallet:activity` |
| State | `SmartWalletState` | **Shipped** | Surface id, preferBlob, last-write note |

`manager-bootstrap.js` (after `app.js`):

- Registers adapters to existing `refreshHistory`, `fetchPrices`, fee-aware quote helpers where available  
- On `tx:finalized` / `tx:confirmed` → light balance refresh (`scheduleBalanceLightBurst` / `refreshBalanceLight`)  
- Notes storage writes for multi-window coordination  

**Important:** Most product code still lives in **`app.js`**. Managers are the modular coordination layer; deep body extraction is ongoing.

### 5.1 Fees (product policy — unchanged)

| Action | Platform fee |
|--------|----------------|
| Internal Jupiter swap | **0.45%** (no LI.FI fee) |
| Internal LiFi EVM swap | **0.45%** + current LI.FI **0.25%** = **0.70%** combined |
| Internal LiFi EVM-source bridge | **0.85%** + current LI.FI **0.25%** = **1.10%** combined |
| Send / external DEX / History view | None |

The atomic verifier compares **only** the encoded Smart Wallet 45/85 bps treasury amount. Combined 0.70% / 1.10% is display-only. Direct 0x / V2 / V3 fallbacks stay disabled. Anti-double-charge ledger remains in app storage (`smart_wallet_swap_fee_paid_v1` and related paths).

---

## 6. Signing & key model

| Type | Secrets location | Password |
|------|------------------|----------|
| Software | Extension vault (AES-256-GCM, PBKDF2 ~650k) | Global ON = user password; OFF = device wrap |
| Ledger | Device only; extension stores addresses/paths | Optional separate global Ledger password |

- Auto-lock via UI + service-worker alarms  
- Offscreen document for local signing helpers  
- `eth_sign` disabled (prefer `personal_sign` / typed data)  
- Free Ledger (password off) can sign while software vault locked  

---

## 7. Live data (idle-first)

| Feed | Behavior |
|------|----------|
| Market WS | Active-chain native ticker only (`live-feeds.js`) |
| Solana activity | `logsSubscribe` mentions wallet address only |
| EVM newHeads | **Not used** (idle spam) |
| HTTP timers | Stop when popup hidden or user leaves Home |

**0.11.34 event path:** confirmed/finalized txs emit `wallet:activity` → light portfolio refresh — **no new background polling loops**.

---

## 8. Storage & multi-window

| Mechanism | Role |
|-----------|------|
| `smart_wallet_v1` (+ legacy `gladiator_*` migration) | Canonical wallet blob |
| Monotonic `rev` + CAS in `storageSet` | Prevents stale window overwrites |
| Dual write chrome.storage + localStorage | Resilience across surfaces |
| `SmartWalletState` surface id / last-write | Ignore stale echoes |

Preference rules unchanged: richer account sets and higher `rev` win.

---

## 9. dApp connect

```text
dApp page → injected provider → content-script → background handler
  → requestUserApproval (wallet UI) → sign → reply
```

Inject hosts: `inject-allowlist.js` (synced into `manifest.json` match lists). Not every `https://*` site.

---

## 10. Build & verification

```text
Load unpacked folder (canonical)
  → tools/verify-extension.ps1   (syntax + version + required files)
  → build-store-package.ps1      (clean zip; includes all manager modules)
```

Store zip **excludes** `.env`, recovery tools, private notes.  
Public GitHub repo remains **docs only**.

---

## 11. File map

| File | Role |
|------|------|
| `manifest.json` | MV3, permissions, content scripts, CSP |
| `chain-registry.js` | Chains + ordered RPC lists |
| `cache-coordinator.js` | Shared cache + inflight |
| `rpc-gateway.js` | Multi-RPC + scoring |
| `rpc-manager.js` | UI RPC client + SW proxy |
| `sw-events.js` | Event bus |
| `state-coordinator.js` | Multi-window helpers |
| `tx-intent.js` | Transaction intent model |
| `transaction-manager.js` | Lifecycle + multi-RPC confirm |
| `portfolio-manager.js` | Staged portfolio orchestration |
| `price-manager.js` | Price façade |
| `history-manager.js` | History façade + durable cache |
| `swap-manager.js` | Swap façade + 0.45% fee math |
| `evm-swap-providers.js` | Sequential Internal DEX quotes (see [INTERNAL-DEX.md](./INTERNAL-DEX.md)) |
| `evm-error-classify.js` / `tx-error-present.js` | Error System (see [ERROR-SYSTEM.md](./ERROR-SYSTEM.md)) |
| `bridge-manager.js` | Bridge façade + 0.85% fee math |
| `manager-bootstrap.js` | Adapter wiring after app.js |
| `app.js` | UI + most product logic |
| `background.js` | SW: dApp, session, RPC proxy |
| `live-feeds.js` | Idle-first WS |
| `fee-helpers.js` | Pure fee bps helpers |
| `offscreen-sign.js` | Local sign helper |
| `inject-allowlist.js` / `content-script.js` / `injected.js` | dApp surface |

---

## 12. Roadmap (honest)

| Priority | Item | Status |
|----------|------|--------|
| — | Chain registry + RPC gateway | **Shipped** |
| — | Cache coordinator | **Shipped** |
| 1 | Transaction intent + lifecycle + multi-RPC confirm | **Shipped** for EVM post-broadcast finalize; full `runLifecycle` UI migration **Partial** |
| 2 | Portfolio / price managers | **Shipped orchestration**; deep extract **Planned** |
| 3 | History manager | **Shipped orchestration** |
| 4 | Swap / bridge managers | **Shipped façades + fees**; execute mostly in app.js **Partial** |
| 5 | RPC provider intelligence | **Shipped** (sequential scoring) |
| 6 | Multi-window state helpers | **Shipped** (+ existing rev/CAS) |
| 7 | Event-driven updates | **Shipped** bus + activity → light refresh |
| next | Migrate remaining send/swap/bridge submits through `runLifecycle` | Planned |
| next | Move heavy balance/history/swap bodies out of app.js | Planned |
| next | Automated integration tests (Ledger, WC, multi-window) | Planned |

**Do not** start another networking redesign. Next value is lifecycle adoption and deeper extracts.

---

## 13. Design principles

1. **Non-custodial** — no Smart Wallet seed server  
2. **Build on the gateway** — do not replace Chain Registry / RPC Gateway  
3. **One request, one healthy provider** — sequential failover only  
4. **Complete transaction lifecycle** — confirm across RPCs; do not trust a single “submitted” hash alone  
5. **Idle-first** — no polling for architecture theater  
6. **Fees only on internal swap/bridge** (Smart Wallet 0.45% / 0.85%; LiFi currently +0.25% on EVM)  
7. **Allowlist inject** — host permission ≠ inject everywhere  
8. **Docs match reality** — never mark shipped without code  

---

## Related documents

| Document | Contents |
|----------|----------|
| [CHAINS.md](./CHAINS.md) | Networks order + Arbitrum / Optimism / Avalanche |
| [ERROR-SYSTEM.md](./ERROR-SYSTEM.md) | Inspect / classify / present / Logs |
| [INTERNAL-DEX.md](./INTERNAL-DEX.md) | In-wallet Swap: sequential quotes, confirm truth, 45 bps |
| [LOADS.md](./LOADS.md) | Network load / idle-first pings / comprehensive RPC counts |
| [BUGS-AND-FIXES.md](./BUGS-AND-FIXES.md) | Bugs vs by-design vs fixed |
| [Chrome-extension-store-for-reviewers/](./Chrome-extension-store-for-reviewers/) | CWS reviewer pack |
| [DOCUMENTATION.txt](./DOCUMENTATION.txt) | Full user guide |
| [EXTENSION-README.md](./EXTENSION-README.md) | Install / package / Helius |

---

*Not financial advice. Cryptocurrency involves risk of loss.*
