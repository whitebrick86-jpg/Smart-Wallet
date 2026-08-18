# Smart Wallet â€” Bugs and fixes (plain language)

**Product:** Smart Wallet (Chrome / Opera browser wallet)  
**Historical audit snapshot:** desktop folder `Gladiator-Wallet-0.6.55` (this fileâ€™s tables through **0.11.159**)  
**Live product:** **0.11.257** â€” see [PRODUCT.md](./PRODUCT.md)  
**Last updated:** 2026-08-13

This list is written for people using or shipping the wallet **and for other AI agents**.  
**Separate real bugs from intentional design** (see labels below).

Full architecture of the two systems this era built:  
**[ERROR-SYSTEM.md](./ERROR-SYSTEM.md)** Â· **[INTERNAL-DEX.md](./INTERNAL-DEX.md)**

---

## How to read this list

| Label | Meaning |
|--------|---------|
| **Bug** | Something is wrong; fix or carefully work around. |
| **Edge case** | Uncommon; can still go wrong. |
| **By design** | Intentional product choice â€” not a coding mistake. |
| **Store / process** | Chrome Web Store / operator checklist. |
| **Fixed** | Corrected in current tree; **reload** the extension. |
| **Cleanup** | Dead code removed; behavior should stay the same. |
| **Natural / infrastructure** | Public RPC, chain rules, thin liquidity â€” not app â€œbugsâ€ per se. |

---

## Canonical folder (agents)

| | |
|--|--|
| **Public repository** | https://github.com/whitebrick86-jpg/Smart-Wallet |
| **Live product** | **0.11.298** (Batch 5 verified) |
| **Worker** | 1.6.0 production Worker (not in this repository) |

---

# Part A â€” By design (not bugs)

| # | Topic | What you see | Why intentional |
|---|--------|----------------|-----------------|
| D1 | Locked / unlocked | Password on â†’ keys encrypted at rest; session unlock until auto-lock | Safer than plain keys all day |
| D2 | Session + auto-lock | Log in once; Settings idle 15mâ€“3d | Normal wallet UX |
| D3 | Wrong password lockout | ~10 tries â†’ wait | Slows guessing |
| D4 | Software vs Ledger password | Two switches | Separate threat models |
| D5 | Platform fee timing | Fee **after** successful **confirmed** internal swap/bridge | Donâ€™t charge failed or still-pending trades |
| D6 | No fee on Send / external DEX | Internal Swap 0.45% / Bridge 0.85% Smart Wallet; LiFi EVM currently +0.25% | Product policy |
| D7 | dApp inject allowlist | Not every website | Security surface |
| D8 | Approve in toolbar wallet | No second permanent window | Multi-step DEX UX |
| D9 | SOL rent residual | Canâ€™t always send 100% SOL | Solana protocol (~0.00089 SOL + fees) |
| D10 | Rematch same-chain only | Uniswap doesnâ€™t die when you view Solana | Avoid cross-family disconnect |
| D11 | Clipboard guard after paste | Not on every keystroke | Less noise |
| D12 | Bridge keeps source chain | No auto-switch to destination | User navigation |
| D13 | EVM balances take time | Discovery = explorer + tokens + prices | Architecture (improvable, not a â€œbroken balanceâ€ bug by itself) |
| D14 | Ledger Blind signing | Heavy contract calls need it | Device security |
| D15 | Free public RPCs | Occasional slow/429 | Infrastructure |
| D16 | Sequential RPC only | One host at a time; scoring reorders, never fans out | Free-tier + fairness |
| D17 | Later-receipt fee waits for unlock | SW can **see** receipt status 1; it cannot **sign** the fee while locked | Keys are not in the service worker |
| D18 | Logs are local | Settings â†’ Logs, last **300** events, never uploaded | Privacy |

---

# Part B â€” Natural / infrastructure (not app defects)

These **feel** like bugs but are mostly environment or market:

| # | Topic | Notes |
|---|--------|--------|
| N1 | Free RPC rate limits (Polygon, RH official, BSC public) | Retries/failover help; not zero-cost infra. Auth/404/DNS hosts are skipped with a long cooldown. |
| N2 | Thin liquidity â€œraise slippageâ€ on real thin pools | Sometimes real price impact. Internal DEX then tries the **next sequential** provider, not a parallel blast. |
| N3 | Explorer lag after swap | New token may appear after logs/Blockscout. Hash-without-receipt is **Submitted**, not Failed. |
| N4 | Chrome Load unpacked path | Wrong folder / stale zip = old code |
| N5 | Pending native reserved | Another mempool tx can reserve ETH/POL/BNB so **this** send is rejected even though the confirmed balance covers it. That is `PENDING_BALANCE_RESERVED`, not â€œempty wallet.â€ |

---

# Part C â€” Actual open bugs / product gaps

| # | Severity | Whatâ€™s wrong | Notes |
|---|----------|--------------|--------|
| **B1** | Medium | Buy with card may not pre-fill address | Onramper key / partner limits |
| **B2** | Medium | Module script fail â†’ feature quiet-breaks | Modules must load before `app.js` |
| **B3** | Process | Stale `dist-store` zip | Rebuild before store upload |
| **B4** | Improved | EVM portfolio felt slow on open | Staged load since 0.11.3; 0.11.146â€“148 cut duplicate History/price work. Full discovery still heavy on free RPC. |
| **B6** | Edge | Two wallet windows race saves | Prefer one UI; rev/CAS + session gen help |
| **B7** | Edge | Stale quote / long wait before bridge/swap | Fresh quote; quoteSeq cancels stale work |
| **B8** | Edge | Other wallets on page confuse picker | Pick Smart Wallet; EIP-6963 session routing 0.11.126 |
| **B9** | Low | dApp UI label lag after rematch | Hard-refresh tab if needed |
| **B10** | Edge | Residual / later-receipt **fee collect** needs unlocked UI | SW probe is read-only. Honest: fee stays `DUE` until unlock. Do **not** move signing into the service worker. |
| **B11** | Low | Logs not wired to every subsystem | Send / Swap / Bridge / Ledger are connected. Gateway failovers, dApp approve, History, vault, quote ancillary are not yet. |

### Closed this era (were listed open in older docs)

| # | Resolution |
|---|------------|
| **B5** | **Fixed.** Software swap confirm used to be weaker than Ledger. **0.11.35** shared `finalizeEvmAfterBroadcast`. **0.11.152** `confirmed` only on receipt status 1. **0.11.155** later-receipt fee due. |

### Store / process

| # | Status | What |
|---|--------|------|
| **S1** | Open | Store screenshots |
| **S2** | Open | Paste listing + privacy into store form |
| **S3** | Improved | `verify-extension.ps1` **PASS** on **0.11.159** |
| **S4** | Open | Rebuild store zip from **this** folder before upload |

---

# Part D â€” Fixed

## New EVM networks (**0.11.156â€“0.11.159**)

Full account: **[CHAINS.md](./CHAINS.md)**. Loads: **[LOADS.md](./LOADS.md)** Â§14.

| Version | What shipped |
|---------|----------------|
| **0.11.156** | **Arbitrum One (42161, ETH).** Registry, sequential RPC, LiFi Internal DEX, Bridge, explorers. Live: `eth_chainId`, USDC decimals 6, LiFi quote. |
| **0.11.157** | **Optimism (10, ETH).** Same shared path. Live: `eth_chainId`, USDC decimals 6, LiFi quote. |
| **0.11.158** | **Avalanche C-Chain (43114, AVAX).** Official C-Chain RPC, Snowtrace, CoinGecko `avalanche-2` / AVAXUSDT. Live: `eth_chainId`, USDC decimals 6, LiFi quote. |
| **0.11.159** | Base-template hooks: USDC seeds, History/RPC prefer, dApp approve + capabilities, preflight AVAX symbol, Onramp, Ledger `chainMeta`. |
| Order | Networks panel = both Bridge dropdowns from `displayChains()`. Dest excludes source, no reorder. |

**Honest gaps:** 0x quotes still `no_backend`. Native Uniswap V3 encode not wired. In-wallet Send / Ledger / Uniswap tab / completed bridge not live-clicked in a browser.

## Audit repairs + extract + later-receipt (**0.11.150â€“0.11.155**)

| Version | Finding | Fix |
|---------|---------|-----|
| **0.11.150** Batch 1 | Software EVM could sign more than once on failover; Jupiter fee convert could start twice; dApp result died with the SW | `signAndBroadcastEvmSoftwareOnce` (same raw only). Jupiter stages `NOT_STARTED â†’ CONVERT_SUBMITTED â†’ â€¦ â†’ PAID / UNCERTAIN` in `smart_wallet_jup_fee_stage_v1`. Durable dApp results `chrome.storage.session` `smart_wallet_dapp_result_v1`. |
| **0.11.151** Batch 2 | Delayed unlock `session.set` could restore secrets after Lock | Monotonic `SESSION_AUTH_GEN` (`smart_wallet_session_auth_gen_v1`). Lock awaits session remove. Stale gen discarded. |
| **0.11.152** Batch 3 | Hash treated as confirmed; provider invented `0x1` | `evmConfirmStateFromOutcome`. `executeSwap` confirmed **only** on receipt status 1. Unknown chain throws. `PROVIDER_HINT_GEN`. |
| **0.11.153** Batch 4 | Global 8s residual cooldown hid other residuals | Per-residual `nextAt` (4s) + `FEE_RESIDUAL_INFLIGHT`. SW alarm `sw-fee-residual`. |
| **0.11.153** Batch 5 | Logs JSON keys; stale read cache; Sui re-sign risk; Bridge status silent | Recursive JSON redaction. `invalidateReadCache` on lock/chain/revoke. Sui execute uses already-signed bytes. `checkPendingBridgeStatus({quiet:true})` on reopen. |
| **0.11.154** Extract 1 | Presentation logic still duplicated in `app.js` | `presentCaughtError`, `formatFriendlySendError`, `solNativeResidualRentMessage` live in `tx-error-present.js`. **No wording change.** |
| **0.11.155** | Internal swap with hash but later receipt never collected 45 bps | `smart_wallet_swap_await_confirm_v1`. SW `eth_getTransactionReceipt` only. Status 1 â†’ fee `DUE`. Status 0 â†’ `NOT_DUE`. Expiry â†’ `UNCERTAIN`. |

**Audited baseline 0.11.149 is preserved** as `SmartWallet-0.11.149-pre-audit-repair`. Do not overwrite it.

## Error System (**0.11.137â€“0.11.145**)

| Version | What was wrong | Fix |
|---------|----------------|-----|
| **0.11.137** | Dead/auth Polygon RPCs (401/403/404/521/DNS) then UI said **not enough POL** | `evm-error-classify.js`. Cleaned host list. Long cooldown on auth/DNS/404. Holdings stops walking on `balanceOf` revert. |
| **0.11.138** | Friendly string â€œBroadcast failed â€” not enough POLâ€ treated as RPC proof; dApp result-replay storm | Insufficient-funds only from numeric proof or the exact Geth phrase. Persist only signing-critical dApp results. |
| **0.11.139** | Ledger cache rebroadcast a **changed amount** | Cache key includes gas + fees; decode signed raw before reuse. Leftover native proof beats RPC text. |
| **0.11.140** | Queued mempool cost shown as empty wallet | `PENDING_BALANCE_RESERVED` vs `INSUFFICIENT_CONFIRMED_BALANCE`. Presentation layer. |
| **0.11.141** | Flattened `.message` dropped nested ethers `queued` / `tx` / `have` | `inspectError` walks cause/info/data + embedded JSON. |
| **0.11.142** | Same signed raw submitted to hosts 0â€“7 after a queued-balance reject | First deterministic pending-reserved **stops** failover. |
| **0.11.143** | Pre-sign `estimateGas` dumped raw RPC; Nano still prompted | Presenter before the device. Intrinsic-cost + queued facts â†’ pending-reserved. |
| **0.11.144** | Nested `info.error.message` never parsed; second lifecycle sentence appended | Nested message is the source. `isPresentedTxError` pass-through. |
| **0.11.145** | No in-wallet diagnostic surface | Settings â†’ Logs (100, local, redacted, non-blocking). |

See **[ERROR-SYSTEM.md](./ERROR-SYSTEM.md)**.

## Responsiveness + load (**0.11.146â€“0.11.149**)

| Version | What was wrong | Fix |
|---------|----------------|-----|
| **0.11.146** | History on every `refreshAll`; fee hydrate blocked first paint; stacked balance refreshes | Skip History unless panel open. In-flight join. Fee hydrates parallel. Memory cache for local tx / history bag. |
| **0.11.147** | Home opened CoinGecko twice; Solana balance forced another price pass | Majors join inflight; `majorsOnly` honored; 8s majors freshness. |
| **0.11.148** | Soft ticks could stampede free RPC | `sw-request-budget.js` per-screen caps (critical send/sign always allowed). Gateway caches `eth_chainId` / `eth_accounts` / `eth_blockNumber`. Residual recovery fire-and-forget. |
| **0.11.149** | Proven unused helpers / preview HTML | Dead-code cleanup only. Behavior unchanged. |

## Internal DEX reliability (**0.11.91â€“0.11.136**)

| Version | What was fixed |
|---------|----------------|
| **0.11.91â€“93** | Shared EVM quote reliability; live rate-limit before confirm |
| **0.11.94â€“98** | Sell after a good LiFi quote; swap-outcome journal; subphase codes; pre-sign sim revert â†’ sequential fallback |
| **0.11.99â€“102** | Bounded BNB connectors; fallback reconfirm; Confirm Updated Route wiring; dead-route loop stop |
| **0.11.132** | Rejected platform fee no longer blocks **external** DEX or Bridge UI |
| **0.11.133â€“136** | Polygon Ledger: one signature; Tenderly not first; gas tip floor; RPC pool |

See **[INTERNAL-DEX.md](./INTERNAL-DEX.md)**.

## dApp / inject (**0.11.113â€“0.11.131**)

| Version | What was fixed |
|---------|----------------|
| **0.11.113â€“125** | External DEX â€œConfirm in walletâ€ with no window; inject identity; Uniswap connection; no extra sign window |
| **0.11.126â€“128** | EIP-6963 session routing; result-dispatch requires content ack; Base `sendRaw` not first on Tenderly public |
| **0.11.129â€“131** | Holdings wiped after reload; DEX token slots reject wallet addresses; faster token lookup after buy / CA paste |

## UI polish + architecture (**0.11.53â€“0.11.72**)

| Area | What was fixed / improved |
|------|---------------------------|
| History colors | Buy/sell/receive/send/bridge/swap row coding (green/red/purple/cyan) in light + dark |
| Receive | Clickable addresses, chain labels, Copy address SVG + orange CTA |
| Accounts seed UI | Outer plate / never-give box / Ledger frame |
| Login / confirm dialogs | Dark chrome in light UI; thin red border |
| Internal DEX light | % hover readable; balance green; swap CTA purple ink |
| Product logo | Central `ui/ui-logo.js` |
| Leading decimal amounts | `.323` â†’ `0.323` |

## P0: False "no hash found" / Send failed after funds actually sent (**0.11.35â€“0.11.36**)

| Issue | Root cause | Fix |
|--------|------------|-----|
| **0.11.36:** UI showed **Send returned no valid transaction hash** after a **successful** broadcast | After send, `executeSend` required **`^0x[0-9a-fA-F]{64}$` for every chain**. Solana base58, BTC hex, and Sui digests **failed the check** | `isValidSendTxIdentity(chain, sig)` â€” chain-specific rules |
| **0.11.35:** EVM "Transaction not found on â€¦ after broadcast" after mine | Post-broadcast timeout treated as failure | `finalizeEvmAfterBroadcast` â†’ CONFIRMING; fail only on receipt status=0 |
| Soft confirm errors as hard failures | Failover / friendly errors | Soft wording; no rebroadcast on lookup miss |
| Lifecycle identity lost | No durable store | `smart_wallet_tx_lifecycle_v1` + `resumePending` |

**Regression tests:** `tools/test-tx-lifecycle.js`.

## Staged EVM balances (**0.11.3**)

| Issue | Fix |
|--------|-----|
| Home waited on Blockscout + logs + many `balanceOf` | Instant **portfolio cache** paint |
| Reopen / chain switch felt empty then slow | **Light path** first: native + known ERC-20 only |
| Full discovery blocked first paint | **Background full**; throttle ~2 min |
| Manual Sync must still find new tokens | Sync uses `full: true` |

## Ledger EVM â€” Permit2 / send / approve / bridge

| Issue | Fix |
|--------|-----|
| Big-int JSON destroyed Permit2 amounts | `parseEip712JsonSafe` / prepare typed data |
| Clear-sign failed on some devices | Hashed EIP-712 fallback |
| Ghost â€œSentâ€ + View tx (hash never on explorer) | Recover `from` matches wallet; multi-RPC raw broadcast; verify before success |
| BSC / Polygon underpriced | Fee floors + simple transfer ~21â€“25k gas |
| UI â€œSentâ€ then â€œSend failedâ€ after balance refresh | Donâ€™t overwrite verified success |

## Robinhood RPC / Polygon Sync / inject

| Issue | Fix |
|--------|-----|
| 404 / Invalid chain / 403 hosts | Removed dead URLs; publicnode + others |
| ethers unknown network 4663 | Explicit `Network("robinhood", 4663)` |
| `syncBusy` forever | Timeouts on RPC/logs/Blockscout |
| four.meme not inject | Added to allowlist + manifest |

## External DEX simulation (â€œraise slippageâ€)

| Issue | Fix |
|--------|-----|
| Stale `eth_chainId` cache | Injected provider awaits live chainId |
| Approve not confirmed â†’ swap STF | Wait after approve on L2/BSC/RH |
| Uniswap blocked from RH | Multi-chain DEX may follow wallet to 4663 |

### User-confirmed after patches
- Internal DEX **buy/sell** on **BNB**, **Polygon**, **Robinhood Chain**
- External DEX **buy/sell** on those chains (Ledger where used)

### Earlier fixed (still in build)
- Modular: fee / address / clipboard / history / onramp
- Session unlock once / auto-lock
- pump.fun / Jupiter / PCS inject
- Wallet rematch same-chain
- Bridge no auto chain switch
- Ledger Solana fee handling (fee-in-swap, Token-2022 / USDCâ†’SOL paths)

---

# Part E â€” Scan notes for agents (2026-08-13)

### Do not â€œfixâ€ these
1. **Sequential RPC** â€” scoring reorders hosts; it does **not** parallel-fan-out.
2. **Sign-once** â€” software and Ledger rebroadcast **the same raw**. Do not add a second sign on failover.
3. **45 / 85 bps** â€” product constants in `fee-helpers.js`.
4. **0.11.149 audited backup** â€” leave it on disk.
5. **Extraction Batch 2** â€” not started; do not start unless asked.
6. **Public GitHub** â€” docs only. Never push extension source, vaults, or `.env`.

### Footguns
1. **`dist-store/package`** is a snapshot. Edit the live folder, then rebuild.
2. **Sibling desktop folders** are not the live extension.
3. Residual / later-receipt **collection** needs an unlocked popup. SW probe is read-only on purpose.

### Verify
```
powershell -File tools\verify-extension.ps1
```
Expected: **VERIFY OK - Smart Wallet 0.11.159** (or current `manifest.json` version).

---

# Part F â€” What to do next

1. Always load **`Gladiator-Wallet-0.6.55`** (reload after pulls/edits).
2. Rebuild store zip before Chrome upload.
3. Store screenshots + listing.
4. Optional: wire remaining Logs subsystems (gateway, dApp, History) â€” report-only until asked.
5. Optional: Multicall3 batch `balanceOf` + optional paid RPC in Settings.
6. Smoke: Polygon/BNB send (pending-reserved copy if a tx is queued); internal swap hash-then-later-receipt still collects **once**; external DEX still has **no** platform fee.

---

## Related docs

| File | Role |
|------|------|
| [CHAINS.md](./CHAINS.md) | Networks + Arb / OP / Avalanche |
| [ERROR-SYSTEM.md](./ERROR-SYSTEM.md) | Error System architecture |
| [INTERNAL-DEX.md](./INTERNAL-DEX.md) | Internal DEX architecture |
| [LOADS.md](./LOADS.md) | Ping / HTTPS / RPC counts |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Build shape |
| [DOCUMENTATION.txt](./DOCUMENTATION.txt) | User guide |
| [CHROME-WEB-STORE-READINESS.md](./CHROME-WEB-STORE-READINESS.md) | Store checklist |

---

*Not financial advice. Cryptocurrency involves risk of loss.*
