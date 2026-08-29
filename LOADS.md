# Smart Wallet — Network Loads (Pings, RPC, APIs)

**Product:** Smart Wallet (Chrome / Opera MV3)  
**Code snapshot:** load-count **0.11.159** · **Live product:** **0.11.660** ([PRODUCT.md](./PRODUCT.md))  
**Last updated:** 2026-08-29 (totals + 1…1M user scale + cap solutions)  

**Live scan (0.11.660):** max-use numbers in **§0** are a code-read of the live unpacked tree (`manifest.json` **0.11.660**, last committed **0.11.657**). Not a Chrome Network HAR. Typical vs MAX envelopes. Failover multiplies **tries**, not logical rounds, except the 450 ms delayed hedge which can fire host #2 in parallel.

**Architecture:** Chain RPC goes through **rpc-gateway** (UI + service worker): sequential multi-RPC failover, per-host cooldown after 429, hard timeouts (`eth_getLogs`), in-flight dedupe, provider scoring. **No free-RPC fan-out.** Host lists: **chain-registry**. Popup Solana/EVM JSON-RPC can proxy via the service worker (`smart-wallet-sol-rpc` / `smart-wallet-evm-rpc`); if that transport fails the popup **replays** the same walk (failed path ≈ 2×).

**0.11.656–657 delayed hedge:** account-family reads (`getBalance`, Solana token accounts, gateway EVM account reads, BTC/Sui REST) start host #2 after **450 ms** if host #1 is still in flight. Happy path **+0**. Slow first host **+1**. Both fail, then the rest of the sequential walk still runs. Home EVM `eth_getBalance` / Multicall use **raw `fetch`** and **do not** take this hedge (or the global inflight 8).

**0.11.416+ Solana ALT verifier:** If the local Solana.com + PublicNode lookup-table quorum is unavailable, the extension may POST only ALT public keys (`MAX_KEYS = 8`) to `smart-wallet-solana-alt-verifier.smart-wallet.workers.dev`. Not a general RPC proxy.

**Load controls (do not remove):** request budgets, gateway read cache, price-path inflight join, History skipped unless the panel is open, deterministic rejects stop host walks, later-receipt **receipt-only** probes, residual-fee alarm. See [ARCHITECTURE.md](./ARCHITECTURE.md).

**How to read a “count” in this file**

| Word | Means |
|------|--------|
| **Ping** | One logical outbound attempt (HTTPS REST **or** JSON-RPC HTTP). Soft-live “rounds” are pings. |
| **HTTPS** | REST to aggregators / prices / explorers / WC / Onramper (not chain JSON-RPC). |
| **RPC** | Chain JSON-RPC (`eth_*`, Solana, Sui, BTC HTTP APIs treated as chain reads). |
| **WS** | Long-lived WebSocket (Binance ticker, Solana `logsSubscribe`). Not counted as a REST ping. |
| **Round** | One logical job. Failover may multiply **RPC tries** (1–4 hosts) but is still one round. Sequential only. |

**Not loads:** theme CSS, product logo, auto-lock timers, vault crypto, Settings → Logs (local `chrome.storage` only). Local diagnostic writes (`smart_wallet_diag_logs`) are **not** network requests. RPC host-failure and recovery rows are recorded fire-and-forget from the existing sequential walk — they do **not** add RPC calls, do **not** retry a successful host, and do **not** change request frequency. The 500/1,000-row Logs cap is local storage only.

---

## 0. Live scan — 2026-08-29 (product 0.11.660)

Code-read of the live unpacked wallet. **Not** a Chrome DevTools HAR. Numbers are **caps and typical envelopes**.

**Round** = one logical job (one balance, one quote, one History fetch). Failover may add **tries**. Hedge may add **1 parallel try**. `<img>` logos are extra HTTPS, counted separately.

### 0.1 Hard request ceilings

| Control | Live value | Where / meaning |
|---------|------------|-----------------|
| Global inflight | **8** | `rpc-active-network.js` `GLOBAL_MAX` — gateway/automatic chain work |
| Per-chain inflight | **4** | `PER_CHAIN_MAX` |
| Networks picker | **2** concurrent | `PORTFOLIO_MAX`; lease 45 s; cancelled on close |
| Screen budget (non-critical) | Home **16** / Send **20** / Swap **24** / Bridge **24** / History **12** / Holdings **16** / dApp **20** / Settings **4** / Logs **2** per **15 s** | `sw-request-budget.js`. Critical send/swap/sign/broadcast/preflight/confirm **bypass**. **Only Home live-balance currently calls `allow()`** — other screens are not actually gated by this table |
| RPC walk | Sequential; Solana **≤7 hosts / 3 attempts**; EVM gateway **≤3**; raw EVM `eth_call` **≤4** | No `Promise.all` fan-out of free RPCs |
| Account-read hedge | **450 ms** | Host #2 starts if host #1 still pending (`rpc-gateway.js`, BTC/Sui in `app.js`) |
| Confirm (EVM) | Poll **~0.4–1.5 s**, overall **~90 s**, **maxHosts 2–5** (BSC **2**) | Then stop; timeout stays **pending**, not failed |
| Confirm (Solana) | **≤16** `getSignatureStatuses` (`default 8`) | Shared `waitSolConfirmed` |
| Price batch | **≤50** mints / Jupiter Price v3 | One HTTPS round |
| EVM DexScreener enrich | **8**/batch, **24** mints, parallel **1** | `evm-holdings-enrichment.js` |
| Logo fallback | **12** cap, concurrency **3** | Then allowlisted CDN `<img>` |
| EVM unknown-mint probe | **≤32** / pass, **5** workers | First-open / full Sync only |
| History | Helius **20** incremental / **50** full **or** RPC **20** / **60** sigs in batches of **5** | **0** unless History is open (or 60 s job while that panel is showing) |
| Mail pull | **≤12** addresses sequential; cooldown **15 s** (force Refresh bypasses) | Sends **40 / hour** (server 429) |
| Announcements | **1 GET**, cache **12 min** | Stops when hidden or vault locked |
| Swap quote debounce | **180 ms** | Cache **8 s**, max **16** entries, auto-refresh **≤2** |
| Bridge LiFi status | **10 s** (Solana source **12 s**), **≤50** tries (Sol **90**) | Per bridge, not idle |
| Jobs | Pause when locked / offline / hidden | Resume stagger **140 ms** |

Idle Home timers: price **90 s**, balance **105 s**, Solana+WS safety **4 min**, sticky native skip **25 s**. Hidden popup/tab **stops** those loops and both WebSockets.

**Gating caveat:** Home EVM native + Multicall + mint probes use raw `fetch()`, so they can run **outside** the global inflight **8**. dApp `eth_call` is also not budget-gated (only 8/4 inflight). Solana dApp RPC (`injected` → background allowlist) walks **5** public hosts and **bypasses** `GLOBAL_MAX`.

### 0.2 Max usage by surface (one action)

**11 display chains:** Solana, Ethereum, Bitcoin, Polygon, Sui, Robinhood, Base, BSC, Arbitrum, Optimism, Avalanche.

| Surface | Typical HTTP/RPC rounds | MAX one action | Extra / notes |
|---------|-------------------------|----------------|---------------|
| **Open wallet (Home, Solana, known bag)** | **6–9** + **2 WS** | **~20–40** | `getBalance` + 3 token-account scans + Jupiter price + CoinGecko majors + 0–1 sparkline + 0–1 announcements GET. WS: Binance `solusdt` + Solana `logsSubscribe` |
| **Open wallet (Home, EVM, known tokens)** | **4–8** + **1 WS** | **~15–25** | Native `eth_getBalance` + Multicall3 + CoinGecko + 0–3 DexScreener batches. After **120 ms**, a **background full discovery** may start if last full ≥ **2 min** |
| **Open wallet (first EVM / empty bag / Sync)** | **40–80** in ~30 s | **~120** (timeout-bound) | Blockscout pages (≤4; Polygon 2) + `eth_getLogs` chunks + ≤32 mint probes. Theoretical 400+ `eth_call`s is **not** reached; wall clock **~22–35 s** |
| **Open wallet (BTC)** | **2–4** | **~6** | Blockstream + 450 ms hedge to mempool.space + CoinGecko + sparkline |
| **Open wallet (Sui)** | **2 + N coins** | **~(2+N)× hedge** | `suix_getBalance` + `suix_getAllBalances` + metadata per coin |
| **Networks tab open** | **~24–30** | **~80–150** if hosts fail | **11** natives + token USD (EVM Multicall, Solana 3 SPL scans) at concurrency **2**, then **cancel on close**. 45 s skip if already fresh. Optional CoinGecko if majors > **2 min**. Does **not** run Blockscout / logs |
| **Switch network** | **2–6** + WS reconnect | **1 full discovery** of the **new** chain | Paints 60 s portfolio cache first. Identity prewarm **1** `eth_chainId` / `getVersion` (3.5 s, 60 s/host throttle). **Not** an 11-chain fan-out |
| **Trending / Discover open** | **Solana 4** / **EVM 3** JSON + site/token imgs | **Solana 7** / **EVM 6** JSON + **~20–48 imgs** | Parallel: Worker `/v1/trending` + CTO (2) + Pumpfun (Solana). Worker miss → DexScreener `token-profiles/latest/v1` + `token-boosts/top/v1` + tokens batch. Cache **3 min**. Bitcoin Discover JSON **0** (sites only). Sites use DuckDuckGo IP3 |
| **Latest CTOs** | **2** | **2** | DexScreener `community-takeovers/latest/v1` + `latest/dex/tokens/{batch}`. Cache **3 min**. Chain-scoped |
| **Pumpfun Movers (Solana only)** | **1** + ≤10 imgs | **1** | Market-data `GET /v1/pumpfun/trending?limit=40`, then filter **$25k mcap AND $5k volume**, show **10**. Hidden on non-Solana. Cache **60 s** |
| **Discover search** | **1** after **350 ms** | **2** (address path) | Name: DexScreener `latest/dex/search`. Address: tokens then `token-pairs/v1/{chain}/{q}` |
| **History (Solana, Helius key)** | **1** | **1** + optional mint labels | Enhanced `limit=20` incremental / **50** full. Cache **45 s**. Incremental if durable rows exist. Unknown mints may then hit Jupiter search (≤40) |
| **History (Solana, public RPC)** | **21** incr / **61** full (1 sig list + `getTransaction` batches of 5, 80 ms pause) | **~21–61** + failover + mint labels | **20** or **60** sigs. `resolveMintMeta` worst **≤40 × 3 HTTPS** if labels are cold |
| **History (EVM, reopen / tip)** | **~20–80** RPC | log chunks **≤6** × hosts + native blocks + ≤18 tx hashes | lookback hundreds of blocks, `maxChunks 3`, concurrency **2**. Blockscout skipped when durable rows exist |
| **History (EVM, full / empty store)** | **~80–200** in the wall-clock window | coded ceiling is higher (`maxChunks` 16–20, native `getBlockByNumber` cap, ≤48 tx hashes, ≤20 timestamps) + **2** Blockscout | ETH lookback **4500** / chunk **220**. UI keeps **60** rows. 60 s auto-refresh **only while History is open**. Do not plan on 400+ completing — deadlines **8–20 s** stop the walk |
| **History (BTC / Sui)** | **1–4** explorer/RPC | **~8** | Address txs APIs |
| **Send (preflight, no broadcast)** | **3–8** | **~12** | Balance, nonce/`getLatestBlockhash`, fee/gas, optional simulate. Screen budget **20/15 s** but send is **critical** (bypass) |
| **Send (broadcast + confirm)** | **+1 submit + 8–16 confirm** | EVM **~8–40** receipts; Solana **≤16** | Same signed raw on failover. Timeout = pending |
| **Receive panel** | **0** | **0** | Address + QR are local. Copy uses clipboard, not HTTP |
| **Receive → Buy** | Onramper **iframe** | 1 widget session | `buy.onramper.com` only when Buy is used |
| **Native sparkline (Total Balance)** | **0–1** | **1** | CoinGecko `market_chart` days=1, TTL **15 min**, inflight join |
| **Open token chart** | **1–2** | **8** | Native: CoinGecko `market_chart` + `/coins/{id}`. Token: CG contract chart → GeckoTerminal pools/token/OHLCV (≤4 pools) → DexScreener meta. Cache **60 s**. Changing **1H/1D/1W/1M** can fire another bundle |
| **Manual Sync** | Same as a full Home refresh | EVM full discovery MAX | Forces prices + balances; skips staged light path |
| **Messaging Inbox/Sent open** | **1–2 POST /v1/mail/pull** | **12** (one per signed address) | 15 s cooldown; Refresh `force` bypasses. Extra 2.5 s join on paint |
| **Home unread chip** | **0–2 pulls** | same as Inbox | `paintMessagingBadges` → `messagingMergeRemote(false)` — **does** hit the Worker even if Messaging is closed |
| **Announcements** | **0–1 GET** | **1** | 12 min cache + interval while visible and unlocked |
| **Send / reply a message** | **1 POST /v1/mail** | **1** | Server **40/hour**. Single-flight `relayBusy` |
| **Block / report / delete-for-me** | **1 POST** each | 1 | Blocked list fetch when that pane opens (not polled) |
| **Enable / Revoke Ledger mail** | **1 POST** authorize or revoke | 1 | 15 s timeout. Production Worker |
| **Owner reports / broadcast** | **1** list (limit 25) | pagination clicks | Unpacked owner tools only; stripped from store zip |
| **Swap quote (LiFi or Jupiter)** | **1** after debounce | many while typing | Debounce **180 ms** (was ~450). Cache 8 s. LiFi via production Worker only — **no `li.quest`** |
| **Swap execute** | quote/build + send + fee legs + short polls | **~10–25** + confirm | Atomic 45 bps. Post-tx light balance waves |
| **Bridge execute** | quote + send + fee + status | status **≤50–90** | 10–12 s interval. 85 bps fail-closed |
| **dApp EVM (Uniswap-class)** | 1 RPC per `eth_call` / estimate / logs | **8 inflight / 4 per chain** | `eth_call` **not cached**. Can saturate that cap for as long as the tab is connected |
| **dApp Solana RPC** | 1 method / 5-host walk | **outside GLOBAL_MAX** | Allowlisted methods only |
| **WalletConnect** | **0** unless paired | relay WS + HTTPS while session lives | Needs user Project ID |
| **Token logos (after holdings paint)** | **0** if cached | **≤12** fallbacks conc **3** + 1–3 list fetches | Uniswap / 1inch / LiFi token maps; CDN imgs (Jupiter, CoinGecko, IPFS gateway, DuckDuckGo IP3 for dApp favicons) |
| **ALT verifier** | **0** | **1 POST / pubkey** | Only if local ALT quorum fails on a Solana tx |
| **Managed RPC Worker** | **0** store / unenrolled | **1 POST /v1/rpc per logical call** | `production-managed` **cannot activate** in this build. Unpacked enrolled + `FULL_SERVICE` only |
| **Settings / Logs / address book / privacy / session / vault** | **0** | **0** | Local storage / WASM. Diagnostics: **no network, no telemetry** |
| **Popup closed, no due fee/confirm** | **0** | **0** | SW alarms local unless a residual or swap-await row is due |

### 0.3 Combined max-use scenarios (planning)

Assumptions: one account, popup **visible**, healthy hosts unless stated. Hedge + failover can multiply the **try** column.

| Scenario | Logical rounds | Tries (failover ×1–4 + hedge) | What is included |
|----------|----------------|--------------------------------|------------------|
| **Idle Home 1 hour, Solana, WS healthy** | **~45–80** | **~60–160** | Price 90 s, balance ~4 min, 0–5 announcement GETs, 0–4 Home badge pulls if session can sign. **2 WS** |
| **Idle Home 1 hour, EVM / WS down** | **~80–130** | **~120–350** | Price 90 s + balance 105 s + majors |
| **Open → Networks tab → switch 5 chains → Discover (Solana) → History → 1 send → 1 receive → 3 token charts → Inbox** | **~80–160** | **~120–400** | One of each user action in this table. Receive = **0**. Charts mostly CoinGecko. Inbox 1–2 pulls |
| **Heavy trading day (§6 C updated)** | **~700–1,800** | **~1.2k–3.5k** | 4 h Home + 40 quotes (180 ms debounce) + 15 swaps + 5 bridges (status-heavy) + 20 History + 10 Syncs + messaging 40 sends + dApp **not** included |
| **Stress hour: spam Sync + History + quotes + Inbox Refresh + Discover** | **~400–800** | **~1k–2.5k** | Quotes unbounded by screen budget (critical). Inbox Refresh bypasses 15 s. Still capped by inflight 8 on gateway paths |
| **dApp Uniswap 1 hour on the connected chain** | saturates **4/chain** inflight | **thousands of `eth_call`** | Dominates every other wallet surface. Plan RPC limits around **this**, not idle Home |
| **Popup hidden** | **0** | **0** | Except due swap-await / fee-residual SW probes |

**Operator rule of thumb (0.11.660):**

- Idle visible Home: **~50–90 HTTP rounds/hour**
- Power user, no dApp: **~1,000–2,000 rounds/day**
- Uniswap/Jupiter **in-page dApp**: treat as **continuous RPC** at **4 inflight / chain** for the whole session
- Messaging: **~5 announcement GET/hour** + **≤40 sends/hour** + pulls **~4–8/hour** typical, **~240 merge rounds/hour** only if Inbox stays open and keeps painting
- **~3,000+ tries** is user-driven stress or a dApp tab, not the idle product

Typical payload **per** request (unchanged class):

| Kind | Request | Response |
|------|---------|----------|
| JSON-RPC read | **0.2–1 KB** | **0.5–8 KB** (getLogs chunks larger but bounded) |
| Jupiter price batch | **~0.5 KB** | **2–20 KB** |
| Jupiter / LiFi quote | **1–4 KB** | **5–50 KB** |
| Mail pull/send | **0.5–2 KB** | **1–20 KB** |
| Solana signed tx | **≤1.3 KB** | small sig |
| Icon / chart | **0.5 KB** | **5–80 KB** |

Idle hour on the wire: **~0.1–2 MB**. Hard trading day: **~5–30 MB**. A busy dApp hour can exceed that in JSON-RPC alone.

### 0.3a Total pings per person (RPC vs HTTP vs Worker)

**Ping** = one logical outbound HTTP attempt (JSON-RPC **or** REST). WebSockets are listed separately and are **not** added into Total pings.

Three planning personas (healthy hosts, one account, popup **visible**). Mid of the §0.3 ranges.

| Bucket | What it is | **Idle Home 1 hour** (Solana + WS) | **Typical day** (~2 h Home, 1 Discover, 1 History, 0–1 send, a few messages) | **Heavy trading day** (4 h Home, quotes + 15 swaps + 5 bridges, no dApp) |
|--------|------------|--------------------------------------|-------------------------------------------------------------------------------|--------------------------------------------------------------------------|
| **Chain RPC** | Solana / EVM / BTC / Sui JSON-RPC or explorer HTTP | **~20** | **~80** | **~600** |
| **Public HTTPS** | Jupiter, CoinGecko, DexScreener, GeckoTerminal, Blockscout, CDN logos | **~20** | **~70** | **~400** |
| **Smart Wallet Workers** | Mail + LiFi proxy + market-data + ALT (all `*.smart-wallet.workers.dev`) | **~10** | **~35** | **~200** |
| **Total pings** | RPC + public HTTPS + Workers | **~50** | **~185** | **~1,200** |
| **WebSockets** | Binance ticker + Solana `logsSubscribe` | **2** sessions | **2** sessions × hours open | **2** sessions × hours open |

Idle hour split (the **~50**):

| Source | Pings |
|--------|-------|
| Solana balance safety (~4 min) | **~15 RPC** |
| Jupiter Price reconcile (often skipped while WS fresh) | **~10 HTTPS** |
| CoinGecko majors / sparkline | **~0–5 HTTPS** |
| Announcements `GET /v1/mail/announcements` (12 min) | **~5 Worker** |
| Home mail-badge pulls (15 s cooldown, unlocked) | **~4 Worker** |
| **Total** | **~50 pings + 2 WS** |

Closed popup: **0 pings** (except a due swap-await / fee-residual). Receive panel: **0**. Settings / Logs: **0**.

Public APIs (Jupiter, CoinGecko, DexScreener, public RPC) are **per user IP**. They do **not** add together on one quota when user count grows. **Workers and LiFi `li.quest` (from the Worker’s IP)** are **shared** — that is what 10 / 100 / 1M people stress.

### 0.3b Scale: 1 → 10 → 100 → 1k → 10k → 100k → 1M people

**N = daily active wallets** (people who actually open the extension that day). Installed-but-closed copies are **0**. If only 10% of N have Home open at once, divide the “all N idle 1 hour at once” column by 10.

#### If all N have Home open for the same idle hour

| People (N) | Chain RPC | Public HTTPS | **Workers (shared)** | **Total pings** | WS sessions |
|------------|-----------|--------------|----------------------|-----------------|-------------|
| **1** | 20 | 20 | **10** | **50** | 2 |
| **10** | 200 | 200 | **100** | **500** | 20 |
| **100** | 2,000 | 2,000 | **1,000** | **5,000** | 200 |
| **1,000** | 20,000 | 20,000 | **10,000** | **50,000** | 2,000 |
| **10,000** | 200,000 | 200,000 | **100,000** | **500,000** | 20,000 |
| **100,000** | 2,000,000 | 2,000,000 | **1,000,000** | **5,000,000** | 200,000 |
| **1,000,000** | 20,000,000 | 20,000,000 | **10,000,000** | **50,000,000** | 2,000,000 |

#### If those N are typical-day DAU (not all open at once)

| People (N) | Chain RPC / day | Public HTTPS / day | **Workers / day (shared)** | **Total pings / day** | Workers / month |
|------------|-----------------|--------------------|----------------------------|-----------------------|-----------------|
| **1** | 80 | 70 | **35** | **185** | ~1k |
| **10** | 800 | 700 | **350** | **1,850** | ~11k |
| **100** | 8,000 | 7,000 | **3,500** | **18,500** | ~105k |
| **1,000** | 80,000 | 70,000 | **35,000** | **185,000** | ~1.1M |
| **10,000** | 800,000 | 700,000 | **350,000** | **1.85M** | ~11M |
| **100,000** | 8M | 7M | **3.5M** | **18.5M** | ~105M |
| **1,000,000** | 80M | 70M | **35M** | **185M** | ~1.05B |

Heavy-trading DAU is ~**6×** the typical Worker column (≈200 Worker pings/user/day). A Uniswap tab is **not** in these tables; it is extra public RPC on **that user’s IP** (thousands of `eth_call` / hour), not extra Worker load.

#### When the shared backends cap

| People (typical DAU) | Cloudflare Workers Free **100k/day** | CF Workers Paid **10M/mo** then $0.30/M | LiFi **unauth** 75 quotes / 2 h (Worker IP) | LiFi **API key** default **100 RPM** (~12k / 2 h) | Jupiter (browser, per IP) | CoinGecko (browser, per IP) | Public RPC (per IP) |
|----------------------|--------------------------------------|------------------------------------------|---------------------------------------------|---------------------------------------------------|---------------------------|-----------------------------|---------------------|
| **1** | OK | OK | OK for light quoting | OK | OK idle; typing quotes can 429 keyless **30/min** | OK idle; chart spam can 429 **10–30/min** | OK in-wallet; dApp can 429 |
| **10** | OK | OK | Tight if several people quote at once | OK | per-IP, still OK | per-IP, still OK | per-IP |
| **100** | OK (~3.5k Worker/day) | OK | **Need LiFi API key** | OK | per-IP | per-IP | per-IP |
| **1,000** | **Near/over Free day** if many wallets stay open (announcements+mail). Typical DAU 35k/day still under 100k | OK | **LiFi key required** | OK unless a quote storm | per-IP | per-IP | per-IP |
| **10,000** | **Free Workers fail** | ~11M/mo → **just over included 10M** (~$5 + $0.30) | **LiFi key**; watch 100 RPM | Raise LiFi plan if >1% quote at once | per-IP | per-IP | per-IP; start **paid RPC** for in-wallet if public 429s rise |
| **100,000** | Fail | ~105M/mo Worker ≈ **$34** requests + $5 | **LiFi enterprise / higher RPM** | 100 RPM is ~1% concurrent quoters | Still per-IP | Still per-IP | **Dedicated RPC** (Helius/Alchemy/Ankr paid). Turn on **production-managed RPC** |
| **1,000,000** | Fail | ~1.05B/mo Worker ≈ **$315** requests + CPU. Cache harder or it is more | **LiFi enterprise + SLA** | Required | Optional **Jupiter paid key** if you later proxy quotes | Optional **CoinGecko Basic/Analyst** if you later proxy prices | **Paid multi-region RPC**. Do not run 1M users on `api.mainnet-beta.solana.com` |

Cloudflare Workers Paid (public 2026-08-28): **$5/mo**, **10 million requests included**, then **$0.30 per extra million**. Subrequests from the Worker to LiFi/Jupiter are **not** billed as extra Worker requests. KV: Free 100k reads/day; Paid 10M reads/mo.

### 0.3c Solutions when amounts cap

Do these in order. None of them require putting API keys in the extension.

| When you hit it | Solution | Notes |
|-----------------|----------|--------|
| Cloudflare Workers **100k/day** (Free) | **Workers Paid** ($5/mo, 10M req/mo) | Needed before ~1k DAU if wallets stay open (mail + announcements). Already implied by live production Workers |
| CF requests beyond 10M/mo | Pay overage **$0.30/M** and **cache** announcements, trending, LiFi tokens at the edge (TTL minutes) | 10k typical DAU is ~11M/mo — cache can keep it in the included 10M |
| LiFi **75 quotes / 2 hours** (no key, Worker IP) | **Free LiFi Partner Portal API key** (`x-lifi-api-key` on the Worker only) → default **100 RPM** (12,000 quotes / 2 h) | Integrator string stays `smart-wallet`. Key never ships in the Chrome zip |
| LiFi 100 RPM not enough | **LiFi paid / enterprise plan** (higher RPM, SLA) — [li.fi/plans](https://li.fi/plans/) | Roughly when **>1% of DAU** quote at the same minute (10k DAU → 100 quotes/min) |
| Jupiter keyless **30/min** on one IP | Keep quotes in the **browser** (today) so each user has their own 30/min; or **Jupiter Free key** 60/min / **Developer $25** 10 RPS if you later proxy via Worker | `lite-api.jup.ag` is being retired; plan `api.jup.ag` + key on a Worker when they cut keyless |
| CoinGecko **10–30/min** or Demo **10k/mo** | Keep majors on **Binance WS** (already). Charts stay on-demand. If you proxy CG, buy **Basic ~$35/mo** (100k credits) or Analyst | One user typical day is tens of CG calls, not thousands |
| Public Solana **~100 req / 10 s / IP** or EVM 429s | Sequential failover (already). Then **Helius Developer $49** / Alchemy / Ankr paid. Last: enable **production-managed RPC** on the RPC-gateway Worker | dApp Uniswap is the first to need this, not idle Home |
| Mail 40 sends/hour / pull storms | Already server-enforced. Raise Worker mail quotas + Durable Object limits on Paid | Do not lift the client 40/hour without a product decision |
| Market-data / Pumpfun / Dex 429 | Worker already aggregates. Add **Cache-Control** / KV TTL (60–180 s) so 10k Discover opens are **1 upstream fetch** | DexScreener pairs **300/min**, profiles **60/min** |
| ALT verifier storms | Rare (fallback only). KV cache verified ALTs | |
| Logos / DuckDuckGo | Already capped 12 fallbacks. Prefer bundled `icons/` | |
| 100k–1M DAU | Paid CF + LiFi enterprise + paid RPC + Worker caches + **do not** put every user on one public RPC URL | Managed RPC (`rpc:use`) is the product path; `production-managed` is still off in this build |

**Do not** put LiFi, Jupiter, Helius, or CoinGecko secrets in `app.js` / `manifest.json` / GitHub. Keys live only in Worker secrets.

### 0.4 Stored payload the wallet can accumulate

Chrome **does not** grant `unlimitedStorage`. `chrome.storage.local` quota is **10 MB**.

| Store | Cap in code | Rough bytes if full |
|-------|-------------|---------------------|
| Logs (`smart_wallet_diag_logs`) | **500** default / **1000** max; message **280** chars | **~0.3–0.8 MB** |
| History UI store | **240** rows | **~0.1–0.4 MB** |
| History manager cache | **40** keys | **~0.5–1.6 MB** |
| Portfolio cache | **60** keys × holdings | **~0.4–1.2 MB** |
| Local tx list | **80** rows | **~40 KB** |
| Tx lifecycle durable | **40** records (100 in memory) | **~20–50 KB** |
| Mail local (`smart_wallet_messages_v1`) | **200** messages; deleted ids **500** | **~0.1–0.5 MB** |
| Token visibility / catalog | grows with discovered mints | **~0.1–0.4 MB** |
| Swap/bridge await + fee residual | 7-day retain | **~50–200 KB** typical |
| Encrypted vault | accounts + settings | **~20–200 KB** |
| Other keys (consent, trusted origins, nonce, UI) | small maps | **~50–200 KB** |

| Envelope | Rough total |
|----------|-------------|
| Quiet single-account wallet | **~0.5–2 MB** |
| Heavy multi-account / many chains / Logs at 1000 | **~3–6 MB** |
| Packed worst before quota pressure | **~8–10 MB** |
| Chrome hard stop | **10 MB** `chrome.storage.local` |

Secrets stay in the encrypted vault blob. Logs, history, mail cache, and portfolio caches must **not** store seed / private key fields.

### 0.5 What this scan did **not** measure

- Live Chrome Network HAR (no DevTools protocol on this pass).
- Production Managed RPC (cannot activate in this build).
- Exact LiFi Partner Portal RPM on the live integrator key (defaults documented as 100 RPM; confirm in the portal).
- CDN `<img>` byte totals (logos vary 5–80 KB).
- Real DAU vs installs — §0.3b treats **N as daily active wallets**.

Historical idle/trading tables in §2–§10 are the evolution story. Use **§0** for current 0.11.660 planning.

---

### Payload endpoints still in use (0.11.660)

| Category | Endpoints (representative) |
|----------|----------------------------|
| **Swap quotes/build (Solana)** | `https://lite-api.jup.ag/swap/v1` · price `…/price/v3` · token search `…/tokens/v2/search` |
| **Bridge / EVM routes** | LiFi **Worker only**: `smart-wallet-lifi-proxy.smart-wallet.workers.dev` `/v1/lifi/quote` · `/advanced/routes` · `/advanced/step-transaction` · `/status` · `/tokens`. No direct `li.quest` |
| **Prices** | Jupiter Price v3, CoinGecko `simple/price` + `market_chart` + `/coins/{id}`, DexScreener token fallback, GeckoTerminal chart fallback |
| **Discover / Trending** | Market-data Worker `…/v1/trending?chain=` · `…/v1/pumpfun/trending?limit=40` · DexScreener `community-takeovers/latest/v1` · `latest/dex/tokens/{batch}` · `token-pairs/v1/{chain}/{q}` |
| **Market WS** | `wss://stream.binance.com` — **1** ticker for the **active** native (SOL, ETH, BTC, BNB, POL, SUI, AVAX). No Robinhood/Base-only ticker |
| **Solana activity WS** | `logsSubscribe` mentions, publicnode / solana.com / drpc |
| **History (optional)** | Helius enhanced (`api.helius.xyz`) when user sets key; else public RPC `getSignaturesForAddress` + `getTransaction` |
| **Messaging** | Production RPC-gateway Worker `/v1/mail`, `/v1/mail/pull`, `/v1/mail/announcements`, block/report/delete, `/v1/mail/ledger-authorize` |
| **EVM discovery** | Chain multi-RPC + Blockscout / Snowtrace-class explorers (per chain-registry) |
| **Official extra RPCs** | `arb1.arbitrum.io` · `mainnet.optimism.io` · `api.avax.network/ext/bc/C/rpc` |
| **BTC** | `blockstream.info` / mempool.space APIs |
| **Sui** | public Sui HTTP RPCs from chain-registry |
| **ALT verifier** | `smart-wallet-solana-alt-verifier.smart-wallet.workers.dev/v1/solana/alt/verify` (fallback only) |
| **WC / Onramp** | Reown / WalletConnect relays when user pairs; Onramper when Buy is used |
| **Icons (CDN)** | DuckDuckGo IP3 favicons; CoinGecko / Jupiter / 1inch / IPFS gateway (allowlisted) |

This document is a full outlook of **network load**: what we used to fire, what we fire now, and **maximum-use totals** (order-of-magnitude) so operators can plan free-tier limits and rate-limit risk.

"Loads" here means **HTTP / WebSocket / RPC traffic from the wallet**.  
It does **not** mean Smart Wallet product analytics (there is no proprietary phone-home).

---

## 1. What counts as a load

| Category | Examples |
|----------|----------|
| **Price APIs** | Jupiter Price, CoinGecko, DexScreener fallback, Binance ticker WS |
| **Chain RPC (HTTP)** | `getBalance`, token accounts, `getTransaction`, EVM `eth_call` / logs |
| **Chain RPC (WebSocket)** | Solana `logsSubscribe` (mentions); EVM heads (**removed**) |
| **Aggregators** | Jupiter swap quote/build, LiFi quote/tx |
| **History** | Helius enhanced (optional) or multi-RPC signature + tx fetch |
| **WalletConnect** | Reown relay (wss/https when user pairs) |
| **Not loads** | Auto-lock timers, password crypto, local storage, Ledger HID |

**Important:** Soft live timers and feeds stop when the popup/tab is **hidden**. Leaving Home stops **balance/price HTTP loops**; Swap/Bridge keep **market WS only** for display prices (no quote spam, no balance poll loop).

---

## 2. Previous load model (before idle-first + live-feeds)

Roughly **pre–0.10.0** soft-live behavior (and early live-feeds before idle tuning):

### 2.1 Continuous while Home open

| Load | Cadence | Typical endpoints | Notes |
|------|---------|-------------------|--------|
| Token holdings USD | ~**35s** tick (min ~30s) | Jupiter Price v3, up to **50 mints / batch** | 1 HTTP per tick if due |
| On-chain balances (light) | ~**90s** | Public multi-RPC (try up to ~4) | Always polling |
| Native majors (SOL/ETH/…) | ~**2 min** when needed | CoinGecko | Rare DexScreener fallback |
| Native chart | ~**15 min** cache | CoinGecko market chart | On paint / range |
| PnL snapshot | ≤ ~**10 min** | **Local only** | No network |

### 2.2 Early WebSocket experiment (briefly worse idle load)

| Load | Behavior | Problem |
|------|----------|---------|
| Market WS | Up to **7** Binance tickers at once | Constant stream traffic even when idle |
| EVM `newHeads` | Every new block → balance refresh (throttled ~20s) | **Idle spam** on ETH/Base/BSC |
| Solana mentions | Activity → refresh | OK when idle |

### 2.3 On-demand (same then and now, roughly)

| Action | Loads |
|--------|--------|
| Manual Sync | Force prices + full/light balance refresh |
| History refresh | Helius (≤50 rows) **or** sig list + `getTransaction` in batches of 5 |
| Swap quote | Jupiter or LiFi quote HTTP (debounced on typing) |
| Swap execute | Jupiter/LiFi build + broadcast + **fee convert/transfer** (extra 1–3 txs) |
| Bridge | LiFi quote + send + fee settle + status polls ~10s |
| dApp use | Inject is local; signs may hit RPC / offscreen |

### 2.4 Previous soft-live totals (Home open, Solana, idle)

Assume **1 hour** continuous Home open, **no user actions**, **no price moves**:

| Source | Estimate |
|--------|----------|
| Jupiter price batches | 3600 / 35 ≈ **~103 HTTP** |
| Balance light refreshes | 3600 / 90 ≈ **~40 multi-RPC** (each may try 1–4 endpoints) |
| CoinGecko majors | 3600 / 120 ≈ **~30 HTTP** (if path ran every 2 min) |
| Chart | **0–1** if already cached |
| WS | **0** (before live-feeds) or **high** if 7-ticker + newHeads era |

**Ballpark previous idle hour:** ~**170–250 outbound HTTP "rounds"** (plus RPC failover multiplies).

With **EVM newHeads era** active on a fast chain (~2s blocks) even throttled to 20s: **+180 balance-trigger events/hour** → much heavier.

---

## 3. Mid load model (0.10.x idle-first — superseded by §4)

This was the **first** idle-first design (still quieter than pre-0.10). Kept for comparison.

### 3.1 Soft live while Home open (0.10.x)

| Load | Cadence / rule |
|------|----------------|
| Holdings USD HTTP | Tick **~45s**, min gap **~30s** (Jupiter batch) |
| Balances HTTP | **~90s**, or safety **~4 min** with Solana activity WS |
| Majors HTTP | Min **~2 min**; skip when market WS fresh (esp. non-Solana) |
| Market WS | **1** Binance ticker; UI push if move ≥**~0.2%** and ≥**~12s** |
| Solana RPC WS | Mentions only; activity debounce **~2.5s**, min gap **~20s** |
| EVM RPC WS | **Off** |

### 3.2 Soft-live totals (0.10.x, Home, Solana, idle, 1 hour)

| Source | Estimate |
|--------|----------|
| Jupiter price batches | 3600 / 45 ≈ **~80 HTTP** |
| Balance HTTP | WS path **~15** / no WS **~40** |
| CoinGecko majors | **0–15** |
| WS | **2** long-lived |

**Ballpark 0.10.x idle hour:** ~**90–120 HTTP rounds** + **2 WS**.

---

## 4. Current load model (0.11.0+ — event-driven + cache)

**0.11.660 live numbers are in §0.** This section is the idle-first design. Quote debounce in live code is **180 ms** (not the ~450 ms shown in older rows below).

**Architecture:** WebSocket / event → **cache** → **UI**, with HTTP as **authoritative reconcile**, not continuous UI polling.

```text
HOME                          DEX (Swap / Bridge)
  |                                |
Market WS (1 ticker)          Market WS only (reuse)
  + Solana mentions WS              |
  |                                |  User input
HTTP price reconcile          Debounce ~450ms
  ~60–120s (tick 90s)              |
HTTP balance safety           Quote API (0 if idle)
  ~3–5 min if activity WS live     |
  ~90–120s if WS down         Execute: fresh quote/build
```

### 4.1 Continuous while Home open (current code)

| Load | Cadence / rule | Endpoints | Idle behavior |
|------|----------------|-----------|---------------|
| **Holdings USD (HTTP)** | Reconcile tick **~90s**, min gap **~90s** | Jupiter Price, ≤50 mints | **Skipped** if holdings cache still fresh |
| **Balances (HTTP)** | Fallback tick **~105s** | Chain multi-RPC (≤~4 tries) | Skips if sticky native <25s |
| **Balances w/ Solana activity WS live** | Safety net **~4 min** (3–5 min band) | Same RPC | **Much quieter** if WS is healthy |
| **Majors (HTTP)** | Min **~2 min** | CoinGecko | **Skipped** when market WS / majors stamp is fresh |
| **Market WS** | **1** Binance ticker (active chain only) | `wss://stream.binance.com…` | UI/cache push if move ≥**~0.2%**, coalesce ≥**~2s** |
| **Solana RPC WS** | `logsSubscribe` **mentions your address** | publicnode / custom / etc. | **Silent** until a tx hits you |
| **EVM RPC WS** | **Not used** | — | No every-block spam |
| **On Solana activity** | Debounce **~2.5s**, min gap **~8s** | One **deduped** light balance (+ soft reprice only if price cache stale) | Only on real txs |
| **Chart** | Cache **~15 min** | CoinGecko | Unchanged class |
| **PnL** | Local | — | No network |
| **Stables (USDC…)** | Pinned ~$1 | — | No price ping |

**Live modes (no duplicate timer stacks):**

| Mode | When | WS | HTTP price loop | HTTP balance loop |
|------|------|----|-----------------|-------------------|
| **home** | Home panel visible | Market + Solana activity | Yes (~90s reconcile) | Yes (WS safety / fallback) |
| **market** | Swap or Bridge visible | Market only | **No** | **No** |
| **off** | Other panels / hidden | **None** | **No** | **No** |

Timers / feeds **stop** when:

- Popup closed / document hidden  
- User leaves Home **and** is not on Swap/Bridge (market-only)  
- Full Sync is busy (soft ticks skip)

### 4.2 Live price & balance frequency (current targets)

| Channel | Frequency | Role |
|---------|-----------|------|
| **Market WebSocket** | Continuous stream; UI on meaningful move | Makes native price **feel live** without HTTP |
| **HTTP price reconcile** | **Every ~60–120s** (implemented tick **~90s** + min gap **~90s**) | Authoritative holdings/majors when cache stale |
| **Balance activity (Solana)** | Event → **~2.5s** debounce → light refresh | Immediate after your txs |
| **Balance safety (WS healthy)** | **~3–5 min** (implemented **~4 min**) | Catch missed logs without spam |
| **Balance fallback (no WS / EVM)** | **~90–120s** (implemented **~105s**) | Controlled poll, not uncontrolled retry |
| **UI paint** | Immediate from **cache / WS / sticky** | Does **not** require a new HTTP to redraw |

### 4.3 On-demand (current)

| Action | Network effect |
|--------|----------------|
| **Open Home** | 1 cheap price tick (if due) + start **home** feeds |
| **Open Swap/Bridge** | Start **market** mode only; strip paints from **cache** first |
| **Sync** | Force prices + balance refresh |
| **History open/refresh** | Helius ≤**50** (or **20** incremental) **or** up to **60** sigs + `getTransaction` batches of **5** + pauses |
| **Swap quote** | Jupiter/LiFi HTTP; debounce **~450ms**; **0** while amount empty / idle |
| **Swap execute** | Fresh quote/build + send main + **fee→USDC convert** + **USDC transfer** (extra 1–2 txs + short balance polls) |
| **Bridge** | LiFi + main tx + fee settle + status ~**10s** until done |
| **Swap panel SOL bar** | Cache/sticky first; optional one-shot SOL RPC only if sticky stale |
| **WC pair** | Reown relays while pairing/signing |
| **dApp** | No continuous wallet pings; RPC only for sign/send as needed |

**Market price ≠ DEX quote:** a market WS tick updates display USD only; it does **not** auto-fire Jupiter/LiFi executable quotes.

### 4.4 Fee settlement load (intentional, per trade)

Each successful **internal** Swap/Bridge may add:

| Step | Loads |
|------|--------|
| Fee convert (if not already USDC) | 1+ Jupiter/LiFi quote attempts + 1 swap broadcast |
| USDC balance polls | A few short RPC reads (~6–10 tries, spaced) |
| Treasury transfer | 1 SPL/ERC-20 transfer |

**Not** continuous background load — only per completed internal trade.

### 4.5 Current soft-live totals (Home open, Solana, idle)

**1 hour**, no txs, flat or modest price moves (WS healthy):

| Source | Estimate |
|--------|----------|
| Jupiter price batches | 3600 / 90 ≈ **~40 HTTP** (fewer if skip-when-fresh) |
| Balance HTTP | With activity WS: 3600 / 240 ≈ **~15** safety nets; without WS: 3600 / 105 ≈ **~34** |
| CoinGecko majors | **0–10** (usually skipped while market WS stamps majors) |
| Market WS | **1 long-lived socket**; UI updates on meaningful moves only |
| Solana mentions WS | **1 long-lived socket**, **0** balance calls if no txs |
| Chart | **0–1** |

**Ballpark current idle hour (Solana + healthy WS):** ~**45–70 HTTP rounds** + **2 WS connections**.

**Target design band:** ~**45–85 HTTP/RPC rounds/hour** idle Home.

**vs 0.10.x idle hour (~90–120):** roughly **~35–50% fewer** soft HTTP rounds again.  
**vs pre-0.10 idle hour (~170–250):** roughly **~70% fewer**.

### 4.6 DEX idle vs active (quotes)

| State | Quote HTTP | Extra market HTTP | Extra balance HTTP |
|-------|------------|-------------------|--------------------|
| **Idle** (Swap open, amount empty, no edits) | **0** | **0** if cache/WS fresh | **0** unless SOL sticky cold |
| **Typing** | ~1 per pause after **~450ms** debounce | 0 for quotes | 0 continuous |
| **Execute** | Fresh quote/build + confirm polls + fee path | As needed | Post-swap light refresh waves |

Example: **20** amount edits → about **20** quote requests after debounce — **not** hundreds of timer-driven quotes.

---

## 5. Side-by-side summary

| Load type | Pre-0.10 | 0.10.x | **Now (0.11.0+)** |
|-----------|----------|--------|-------------------|
| Holdings price HTTP | ~35s | ~45s (+ min 30s) | **~90s reconcile** (+ min 90s); skip if fresh |
| Balance HTTP | ~90s always | ~90s, or **~4 min** with Solana WS | **~105s** fallback, or **~4 min** with Solana WS |
| Majors HTTP | ~2 min | ~2 min, skippable when WS fresh | Same class; stronger skip when WS fresh |
| Market WS UI push | None / 7 tickers | 1 ticker, ≥**12s** | 1 ticker, ≥**~2s** coalesce (still ≥0.2% move) |
| EVM block WS | Sometimes | **Off** | **Off** |
| Solana activity | Optional | On Home; min gap **20s** | On Home; min gap **~8s**, debounce **~2.5s** |
| Leave Home | Soft stop | Soft stop | **Market-only** on Swap/Bridge; full stop elsewhere |
| Swap quotes | Debounce ~380ms | ~380ms | **~450ms**; **0** when idle empty |
| Fee traffic | Direct token/SOL | **USDC convert + transfer** | Same (per internal trade) |
| Popup closed | Soft timers stop | Soft timers + feeds stop | Same |
| Idle Home HTTP/hr | ~170–250 | ~90–120 | **~45–70** (target **45–85**) |

---

## 6. Max-use scenarios (full totals)

Assumptions are **upper bounds** for free-tier planning. Real use is usually lower (caching, skip sticky, debounces, idle WS).

### Scenario A — Heavy day, power user (max soft live)

- Wallet open **8 hours** on Home, Solana, visible  
- Activity WS healthy (balance HTTP ~4 min)  
- Price flat most of the time (WS open, few UI pushes)  
- Holdings: 40 tokens → 1 Jupiter batch each **reconcile** tick  

| Load | Calculation | Max ~count |
|------|-------------|------------|
| Jupiter price batches | 8×3600 / 90 | **~320** |
| Balance light (WS path) | 8×3600 / 240 | **~120** |
| CoinGecko majors | 8×3600 / 120 (worst if never skipped) | **~240** |
| Chart fetches | 8×60 / 15 | **~32** |
| Market WS | 1 connection × 8h | **1 session** |
| Solana mentions WS | 1 connection × 8h | **1 session** |
| **HTTP rounds (soft)** | | **~500–700** (typical) · **~700–900** (worst majors) |
| **WS sessions** | | **2** |

**Monthly (22 such days):** ~**11k–20k** soft HTTP rounds — lower than 0.10.x Scenario A.

### Scenario B — Max soft live without activity WS (EVM or WS down)

| Load | 8 hours |
|------|---------|
| Price reconcile @ 90s | 8×3600/90 ≈ **~320** |
| Balance HTTP @ 105s | 8×3600/105 ≈ **~275** |
| Majors worst | up to **~240** |
| **HTTP soft total** | **~600–850** |

### Scenario C — Max trading day (actions dominate)

Assume in **one day**:

- Home open **4 hours** (soft loads ≈ half of Scenario A)  
- **40** Swap quotes (typing) → ~40–80 Jupiter/LiFi quote HTTP  
- **15** successful Solana swaps → each: main swap + fee convert + fee transfer + a few balance polls  
- **5** bridges → each: 2+ LiFi + send + fee settle + status polls (~5–30 polls if slow)  
- **20** History refreshes → Helius path or heavy RPC `getTransaction`  
- **10** full Syncs  
- **2 hours** WalletConnect session  

| Bucket | Rough max HTTP/RPC "rounds" |
|--------|------------------------------|
| Soft live (4h) | ~250–450 |
| Quotes | ~40–80 (interaction-scaled; not timer spam) |
| Swap executes (15) | ~15 main + ~15–30 fee legs + ~50–100 short polls ≈ **~100–150** |
| Bridges (5) | ~50–150 (status heavy) |
| History (Helius path) | ~20 API calls |
| History (RPC-only path) worst | 20×(1 sig list + 60/5 batches) ≈ **~260+** getTransaction groups |
| Syncs (10) | ~20–40 |
| WC | continuous relay while open (not counted as REST) |
| **Day total REST-ish** | **~600–1,000** (Helius history) · **~1,000–1,600** (RPC history worst) |

### Scenario D — Absolute worst hour (stress)

- Home open, Solana, **WS down**, every timer fires, user spams Sync + History + quotes  

| Action | Count |
|--------|-------|
| Price ticks | ~40–60 |
| Balance polls | ~30–50 (with failover ×1–4) |
| Sync spam (e.g. 30) | ~60–120 |
| History full (e.g. 10) RPC path | ~100–200 |
| Quote spam (e.g. 60 typed pauses) | ~60–120 |
| **Hour total (wild upper)** | **~300–550** rounds |

Normal UI debounces and empty-amount quote skip reduce this heavily.

### Scenario E — Idle night (popup closed)

| Load | Count |
|------|-------|
| Soft timers | **0** |
| Live feeds | **0** |
| Background SW | Alarm auto-lock **local**; no price spam |
| dApp session open | Possible SW traffic only if dApp is actively messaging |

---

## 7. Per-feature load cheat sheet (current)

| Feature | Idle | Active use |
|---------|------|------------|
| **Home prices** | Market WS live; HTTP reconcile **~90s** if stale | Force on Sync |
| **Home balances** | Activity-driven; safety **~4 min** or fallback **~105s** | Force on Sync / activity |
| **Live market WS** | 1 socket; UI on meaningful moves | Same on Home + Swap/Bridge display |
| **Solana activity WS** | 1 socket on Home; silent | 1 light refresh per activity burst |
| **Internal Swap (open, no amount)** | **0 quotes** | — |
| **Internal Swap (typing)** | — | 1 quote / ~450ms pause |
| **Internal Swap (execute)** | — | Fresh build + fee USDC path |
| **Internal Bridge** | 0 continuous | Quotes + execute + fee + status polls |
| **History** | 0 | Helius 1 request / refresh, or multi getTransaction |
| **Send** | 0 | 1–few RPC + broadcast |
| **dApp** | 0 continuous from wallet | Sign path RPCs as needed |
| **WC** | 0 unless paired | Relay while session live |

---

## 8. Multipliers that inflate "call count"

1. **RPC failover** — one logical balance may hit **2–4** endpoints on 429/failure.  
2. **History without Helius** — many `getTransaction` calls (batches of 5).  
3. **Fee USDC path** — +1–2 transactions and several balance checks per internal trade.  
4. **Ledger** — extra time → more chance of re-quote / retries.  
5. **Bridge status** — polls ~every 10s until complete (can dominate a slow bridge).

**429 handling:** price APIs use a **single backoff retry** (~2.5s), then keep last cache — no immediate retry storms.

---

## 9. What reduced load (evolution)

| Era | Change | Effect |
|-----|--------|--------|
| → 0.10.x | Price 35s → **45s**; Solana WS + **4 min** balance safety; 1 market ticker; no EVM heads | Idle ~90–120 HTTP/hr |
| → **0.11.0** | Price reconcile **~90s**; skip HTTP when WS/cache fresh; balance fallback **~105s**; activity min gap **~8s**; market UI coalesce **~2s**; DEX **market-only** mode; quote debounce **~450ms**; idle DEX **0** quotes | Idle **~45–70 HTTP/hr** (target 45–85) |
| Trading | Fee always → **USDC** | Extra convert + transfer **per internal trade only** |

---

## 10. Free-tier practicality (order of magnitude)

| Usage | Soft HTTP / month (order) | Notes |
|-------|---------------------------|--------|
| Light (30 min/day Home) | Low thousands | Fine for free Jupiter/CoinGecko |
| Medium (2 h/day) | ~5k–15k soft rounds | Usually fine |
| Max Scenario A daily | ~11k–20k soft rounds | Watch CoinGecko 429s |
| Heavy trading + RPC history | Soft + **transaction fan-out** | Prefer **Helius** for History |

**Recommendation:** Optional Helius for History keeps high-activity users off free RPC `getTransaction` storms.

---

## 11. One-line comparison

**Previously (pre-0.10):** steady ~35s price + ~90s balance HTTP; briefly multi-ticker market WS + EVM block heads.  

**0.10.x:** quieter idle (~45s prices, Solana activity WS, ~90–120 HTTP/hr).  

**0.11.0–0.11.72:** **event → cache → UI** — market WS for live native feel, HTTP price reconcile ~**90s**, balance activity-driven with **~4 min** safety / **~105s** fallback, idle Home ~**45–70 HTTP/hr**, idle DEX **0** quotes.

**Now (0.11.159):** same idle band. Arbitrum / Optimism / Avalanche add **hosts**, not extra idle timers. Avalanche Home uses `AVAXUSDT` instead of `ETHUSDT` (still **1** market WS). CoinGecko majors add `avalanche-2` **inside the same** HTTPS request. Popup closed remains **0** soft HTTPS/RPC unless a later-receipt or residual job is due.

---

## 12. Comprehensive ping / HTTPS / RPC inventory (0.11.159)

Every outbound class the wallet can fire. Counts are **logical pings per event** unless noted. Failover may turn 1 RPC round into **1–4 sequential tries**.

### 12.1 Idle Home (popup visible, no clicks, Solana, WS healthy)

| # | Channel | Kind | Cadence | Pings / hour (typical) | Pings / hour (worst skip-none) |
|---|---------|------|---------|------------------------|--------------------------------|
| 1 | Jupiter Price v3 (holdings, ≤50 mints) | HTTPS | ~90s reconcile; skip if cache fresh | **20–40** | ~40 |
| 2 | CoinGecko `simple/price` majors | HTTPS | ≥2 min; **skipped** while market WS stamp is fresh | **0–8** | ~30 |
| 3 | CoinGecko market chart | HTTPS | ~15 min cache | **0–1** | ~4 |
| 4 | Light balance (`getBalance` / token accounts) | RPC | ~4 min safety when activity WS is live | **12–15** | ~15 |
| 5 | DexScreener token fallback | HTTPS | Only if Jupiter/CG miss | **0–2** | ~6 |
| 6 | Binance ticker | **WS** | 1 long-lived socket | **0 REST** | 0 REST |
| 7 | Solana `logsSubscribe` mentions | **WS** | 1 long-lived socket; silent until a tx | **0 REST** | 0 REST |
| | **Idle Home total** | | | **~45–70 HTTPS+RPC** + **2 WS** | **~85–95** |

EVM Home (no activity WS): replace row 4 with balance fallback ~**105s** → **~34 RPC/hr**, idle band **~60–85**.

### 12.2 Popup closed / other panel (no Swap, no Bridge, no pending jobs)

| Channel | Pings |
|---------|--------|
| Soft price / balance timers | **0** |
| Market WS / Solana mentions WS | **0** (torn down) |
| Settings → Logs | **0** (local storage) |
| Auto-lock alarm | **0** network (local) |
| **Total** | **0** |

### 12.3 Per-screen request budget (0.11.148) — non-critical only

Sliding **15s** window. **Critical** ops (`send`, `swap-exec`, `bridge-exec`, `sign`, `broadcast`, `preflight`, `confirm`, `fee-collect`) are **always allowed** and do **not** consume the cap.

| Screen | Cap / 15s | Typical consumer |
|--------|-----------|------------------|
| Home | 16 | light balance, price reconcile |
| Holdings | 16 | token discovery |
| Send | 20 | fee estimate (non-critical extras) |
| Swap | 24 | quote extras (execute is critical) |
| Bridge | 24 | status extras (execute is critical) |
| History | 12 | pagination / extra rows |
| Settings | 4 | almost nothing |
| Logs | 2 | none (local) |
| dApp | 20 | chainId / accounts reads |
| Wallet (default) | 24 | catch-all |

When the cap is hit, the extra **soft** ping is skipped (last cache stays). This is the main reason idle Home stays in the 45–70 band even if several timers wake together.

### 12.4 Gateway read cache (0.11.148) — RPC avoided

| Method | TTL | When bypassed |
|--------|-----|----------------|
| `eth_chainId` | 8s | `fresh` flag, send / preflight purpose |
| `eth_accounts` | 8s | same |
| `eth_blockNumber` | 1.5s | same |

Invalidated on lock, chain publish, permission revoke, and after a post-tx burst. A dApp that polls `eth_chainId` every 250 ms costs the wallet **~7–8 RPC/min** instead of **~240 RPC/min**.

### 12.5 Internal DEX — quote (user typing)

| Step | Kind | Pings (LiFi wins) | Pings (full sequential walk) |
|------|------|-------------------|------------------------------|
| Debounce | — | 0 until ~**450 ms** pause | same |
| Idle, amount empty | — | **0** | **0** |
| LiFi quote | HTTPS | **1** | 1 (fail / 429) |
| 0x quote | HTTPS | 0 | **0–1** (supported chains) |
| V2 `getPair` / `getAmountsOut` | RPC | 0 | **1–4** sequential |
| V3 quoter `eth_call` (BNB) | RPC | 0 | **1–4** fee tiers until hit |
| BNB connector search (thin pair) | RPC | 0 | **≤ 28** RPC, **≤ 8 s**, **≤ 14** candidates |
| **Typical typed pause** | | **1 HTTPS** | **1–3 HTTPS + 2–8 RPC** |
| **20 amount edits** | | **~20 HTTPS** | **~20–60 HTTPS+RPC** (not hundreds) |

LiFi 429 → cooldown 20–120 s (skip LiFi, do **not** retry-storm). Same quote key joins in-flight (1 ping, not 2). Executable cache 8 s / no-route cache 4 s.

### 12.6 Internal DEX — execute (one successful EVM swap)

| Step | Kind | Pings |
|------|------|--------|
| Fresh quote / build | HTTPS and/or RPC | **1–3** |
| Allowance `allowance` | RPC | **1** (cache 8 s) |
| Approve (if needed) + receipt | RPC | **2–4** |
| `estimateGas` / fee data | RPC | **1–2** (preflight cache) |
| Optional `eth_call` sim | RPC | **0–1** |
| `eth_sendRawTransaction` | RPC | **1** (+ 0–3 failover on timeout only; **same raw**) |
| Receipt wait (popup still open) | RPC | **2–8** sequential |
| Light balance refresh after | RPC | **1–3** |
| **Main-swap subtotal** | | **~10–25** |
| Platform fee (only if receipt **status 1**) | HTTPS+RPC | convert quote **0–2** + transfer **1** + short USDC polls **2–6** ≈ **3–10** |
| **Grand total, confirmed, fee paid** | | **~15–35** |

**Hash, no receipt (0.11.155):** execute stops at Submitted. Fee pings = **0**. Probe table is §12.8.

Failed / reverted main swap: **0** fee pings.

### 12.7 Internal DEX — execute (one successful Solana / Jupiter swap)

| Step | Kind | Pings |
|------|------|--------|
| Jupiter quote + swap build | HTTPS | **2** |
| `sendTransaction` + confirm | RPC | **2–6** |
| Fee convert (if not USDC) | HTTPS+RPC | **2–6** |
| USDC treasury transfer | RPC | **1–3** |
| **Total** | | **~7–17** |

Jupiter stage `CONVERT_SUBMITTED` / `UNCERTAIN` **blocks** a second convert (saves a duplicate 2–6 ping burst).

### 12.8 Later-receipt probe (only while `mainSwap = SUBMITTED`)

SW alarm `sw-swap-await-confirm`. Method: **`eth_getTransactionReceipt` only**. No sign. No `eth_sendRawTransaction`.

| | |
|--|--|
| First probe | **4 s** after persist |
| Backoff | 4s → 8s → 15s → 30s → 60s → **120s** (then stay at 120s) |
| Expiry | **15 minutes** |
| Max RPC per waiting hash | **~11–13** `eth_getTransactionReceipt` if the receipt never appears |
| Typical (receipt in < 30 s) | **2–4** RPC |
| Concurrent hashes | 1 RPC each due row, sequential, one inflight worker |
| Popup closed | **same probe still runs** (read-only). Fee **collect** does **not**. |
| No waiting hash | **0** |

### 12.9 Residual fee alarm (only if a leftover USDC residual exists)

| | |
|--|--|
| Alarm | `sw-fee-residual` |
| Per residual | `nextAt` **4 s**, one inflight runner |
| Popup closed | SW wakes UI; **0** RPC until UI is unlocked and signs |
| Paid / uncertain identity | **0** retries |
| No residual | **0** |

### 12.10 Bridge (internal)

| Step | Kind | Pings |
|------|------|--------|
| Quote (typing) | HTTPS LiFi | **1** / ~450 ms pause; **0** idle |
| Execute | HTTPS+RPC | quote/tx **2** + send **1–4** + fee settle **3–10** |
| Status poll | HTTPS | ~**every 10 s** until done (**5–30** if slow) |
| Reopen Bridge panel | HTTPS | **1** quiet status (`quiet: true`) — no resubmit |

### 12.11 Send (one native or token transfer)

| Step | Kind | Pings |
|------|------|--------|
| Preflight balance / fee / `estimateGas` | RPC | **2–5** (deterministic funds reject **stops** — no 8-host walk) |
| Sign | local | **0** |
| Broadcast | RPC | **1** (+ failover **same raw** only on timeout) |
| Confirm votes | RPC | **2–6** sequential |
| **Total** | | **~5–15** |

Pending-reserved / insufficient-confirmed / nonce / revert: **stop**. That is often **1–2 RPC** instead of **8**.

### 12.12 History

| Path | Kind | Pings per open/refresh |
|------|------|-------------------------|
| Helius (user key) | HTTPS | **1** (≤50 rows; incremental ≤20) |
| RPC-only Solana | RPC | 1 sig list + `getTransaction` batches of **5**, up to ~**60** sigs ≈ **~13+** groups |
| EVM explorer / logs | HTTPS+RPC | chain-dependent; **not** started by `refreshAll` unless History is open (0.11.146) |

### 12.13 dApp / WalletConnect / Buy

| Surface | Idle | Active |
|---------|------|--------|
| Injected provider | **0** wallet pings | `eth_chainId` / `eth_accounts` / `eth_blockNumber` served from **8s / 8s / 1.5s** cache when safe |
| Sign / send from dApp | — | same Send/swap RPC as in-wallet |
| WalletConnect | **0** unless paired | Reown relay WS + HTTPS while session live |
| Onramper Buy | **0** | partner HTTPS when the Buy panel is used |

### 12.14 Holdings / Sync / token lookup

| Action | Kind | Pings |
|--------|------|--------|
| Open Home (cached bag) | — | **0** until first due reconcile |
| Manual Sync | HTTPS+RPC | prices **1–2** + full discovery (Blockscout + logs + `balanceOf`) **tens** on a busy EVM bag — user-initiated |
| Token lookup after CA paste | HTTPS+RPC | **1–3** (0.11.131 faster path; no wallet-address-as-token, 0.11.130) |

### 12.15 What never pings

| Item | Why |
|------|-----|
| Settings → Logs | `chrome.storage.local` only |
| Error System inspect/classify/present | CPU only |
| Auto-lock / session gen | local |
| Product logo / theme | local assets |
| Vault encrypt / decrypt | local |

---

## 13. Worked totals (0.11.155)

### Hour — idle Home (Solana, WS up)

| | HTTPS | RPC | WS | **Pings (HTTPS+RPC)** |
|--|-------|-----|----|------------------------|
| Typical | 25–45 | 12–20 | 2 | **~45–70** |
| Worst (never skip) | ~70 | ~20 | 2 | **~85–95** |

### Hour — popup closed, nothing pending

**0 / 0 / 0**

### Hour — popup closed, **one** submitted EVM swap waiting on a receipt

| | Count |
|--|--------|
| `eth_getTransactionReceipt` | **~8–13** over the hour (backoff → 120 s), then stop at 15 min |
| Fee collect | **0** until unlock |
| HTTPS | **0** |

### Trading day (same assumptions as §6 Scenario C, plus 0.11.155 controls)

| Bucket | 0.11.0-era | **0.11.155** |
|--------|------------|--------------|
| Soft live 4 h | 250–450 | **200–350** (budgets + History skip + price join) |
| 40 quotes | 40–80 | **40–80** (unchanged class; still not timer spam) |
| 15 EVM swaps confirmed | 100–150 | **100–150** + **0–40** later-receipt probes if some receipts lagged |
| 5 bridges | 50–150 | **50–150** |
| 20 History (Helius) | ~20 | **~20** (and **0** from background `refreshAll`) |
| **Day REST-ish** | 600–1,600 | **~550–1,400** typical; RPC-history worst still the heavy tail |

### Multipliers (still true)

1. RPC failover × **1–4** on 429/dead host — **unless** the Error System names a sticky account reject, then × **1**.
2. History without Helius = `getTransaction` fan-out.
3. Fee USDC path = +1–2 txs **per confirmed internal trade only**.
4. Bridge status ~10 s until done.
5. BNB connector search ≤ **28** RPC, then stop.

---

## 14. Arbitrum / Optimism / Avalanche — what they add to load (0.11.156–159)

Adding three EVM nets **does not** triple idle Home. The wallet still talks to **one active chain** plus one market ticker.

| Situation | Extra pings vs 0.11.155 | Notes |
|-----------|-------------------------|--------|
| Idle Home on Solana | **0** | Same 45–70/hr + 2 WS |
| Idle Home on Arb or OP | Same as Base EVM idle | No Solana mentions WS → balance fallback ~**105s** → **~60–85** HTTPS+RPC/hr + **1** ETHUSDT WS |
| Idle Home on Avalanche | Same EVM idle class | **1** AVAXUSDT WS (replaces ETHUSDT, not a second socket). CoinGecko still **1** majors HTTPS (now includes `avalanche-2`) |
| Open Networks picker | **+3 native balance RPC** (one per new chain) | Sequential per chain, not a fan-out of hosts. One-shot on open, not a timer |
| Internal swap quote (Arb/OP/AVAX) | **1 HTTPS** if LiFi wins | Live-probed. Same debounce ~450 ms; **0** while amount empty |
| Internal swap execute | **~15–35** like other EVM | LiFi quote/build + atomic fee verify + send + receipt |
| History on Arb/OP | Blockscout + optional logs | Prefer Tenderly/drpc/official first (publicnode last for topic-only logs) |
| History on Avalanche | Logs + official C-Chain RPC | No Blockscout URL in registry |
| Popup closed | **0** | Same as before |
| 0x quote | **0** | Runtime skip (`no_backend`) — not a hidden extra HTTPS |

Dead hosts that were **not** left in the walk (they would have been wasted failover pings): `arbitrum.llamarpc.com`, `optimism.meowrpc.com`, `rpc.ankr.com/avalanche`, `avalanche.public-rpc.com`.

---

## Related docs

| File | Role |
|------|------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | System design |
| [CHAINS.md](./CHAINS.md) | Arb / OP / Avalanche registry + tokens |
| [ERROR-SYSTEM.md](./ERROR-SYSTEM.md) | Why sticky rejects stop RPC walks |
| [INTERNAL-DEX.md](./INTERNAL-DEX.md) | Quote / execute / later-receipt jobs |
| [DOCUMENTATION.txt](./DOCUMENTATION.txt) §19 | User-facing load pings table |
| [CHROME-WEB-STORE-READINESS.md](./CHROME-WEB-STORE-READINESS.md) | Store readiness |
| [BUGS-AND-FIXES.md](./BUGS-AND-FIXES.md) | Known issues |

---

*Not financial advice. Counts are engineering estimates for capacity planning, not billing guarantees from third parties.*


