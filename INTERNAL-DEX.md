# Internal DEX

**Product:** Smart Wallet (Chrome / Opera MV3 extension)  
**Docs snapshot:** **0.11.257** (quote coordinator history through 0.11.159 still applies)  
**Last updated:** 2026-08-16  
**Repository:** Documentation only — extension source is **not** published here.

This is the full account of the **Internal DEX**: the in-wallet Swap panel that quotes, independently verifies the atomic platform fee, signs once, broadcasts sequentially, and confirms from a real receipt. Smart Wallet’s swap fee is **0.45%**. On LiFi EVM routes LI.FI currently adds a separate **0.25%** service fee (combined **0.70%**). Those fees are in the same source-chain transaction.

It is **not** Uniswap / PancakeSwap / Jupiter **in a browser tab**. Those are **external DEX** sites that talk to the allowlisted injected provider. External DEX pays **no** Smart Wallet platform fee.

---

## 1. What “Internal DEX” means

| Surface | Where the user is | Who builds the swap | Platform fee |
|---------|-------------------|---------------------|--------------|
| **Internal DEX** | Smart Wallet → **Swap** | Jupiter (Solana) or LiFi (EVM) | Smart Wallet **0.45%** atomic; LiFi EVM currently **+0.25%** LI.FI = **0.70%** combined |
| **Internal Bridge** | Smart Wallet → **Bridge** | LiFi (EVM source) | Smart Wallet **0.85%** atomic; currently **+0.25%** LI.FI = **1.10%** combined |
| **External DEX** | uniswap.org, pancakeswap.finance, jup.ag, … | The site | **None** from Smart Wallet |
| **Send** | Smart Wallet → **Send** | Wallet | **None** |

Internal DEX is a **same-chain swap**. Bridge is a different product path that shares fee helpers and the Error System, not this quote coordinator.

---

## 2. Runtime shape

```text
Swap panel (app.js + swap-manager façade)
        │
        │  amount debounce ~450 ms; quoteSeq cancels stale work
        ▼
  Quote coordinator  ── sequential, never parallel ──
        │
        ├─ Solana  → Jupiter lite-api (quote + later swap/build)
        │
        └─ EVM     → LiFi staging Worker only
                       │
                       └─ HTTPS Worker /v1/lifi/quote (then routes if needed)
                          0x / official V2 / official V3 stay listed but DISABLED
                          (no silent fallback, no fee-free path)
        │
        ▼
  Preflight caches (gas / fee data / sim / allowance / SOL rent)
        │
        ▼
  User Confirm  →  sign-once (software or Ledger)
        │
        ▼
  Sequential broadcast of THAT raw only
        │
        ▼
  Confirm truth
        ├─ receipt status 1  →  CONFIRMED  (Smart Wallet + provider fees already in the signed tx)
        ├─ receipt status 0  →  FAILED     (neither the swap nor those encoded fees complete)
        └─ hash, no receipt  →  SUBMITTED  →  later-receipt probe
                                              (SW eth_getTransactionReceipt only)
```

**Managers never hold keys.** Signing is software vault JIT / Ledger HID / offscreen helper. The quote module never sees a seed.

---

## 3. Modules

| Module | Global | Role |
|--------|--------|------|
| `swap-manager.js` | `SmartWalletSwap` | Façade. **45 bps** via `FeeHelpers`. Anti-double-charge adapters live in `app.js`. |
| `evm-swap-providers.js` | `SmartWalletEvmSwap` | Sequential EVM quote coordinator. Official routers only. |
| `swap-preflight.js` | short-lived caches | Gas / fee-data / sim / allowance / SOL rent / fee-convert quotes. Fingerprints only. |
| `swap-outcome.js` | `SmartWalletSwapOutcome` | State machine + swap codes + privacy-safe DevTools journal. |
| `evm-revert-decoder.js` | (swap-outcome only) | Allowlisted revert selectors. No user copy. |
| `fee-helpers.js` | `FeeHelpers` | Pure math: swap **45** bps · bridge **85** bps. |
| `tx-error-present.js` | `SmartWalletTxPresent` | Broadcast / funds / RPC wording (see [ERROR-SYSTEM.md](./ERROR-SYSTEM.md)). |
| `app.js` | execute / fee ledger | Execute, Jupiter fee stages, later-receipt queue, history row. |
| `background.js` | SW probe | `eth_getTransactionReceipt` + `sw-swap-await-confirm` alarm. **No sign. No rebroadcast.** |

---

## 4. Product policy (unchanged)

| Rule | Value |
|------|--------|
| Internal swap fee | **45 bps = 0.45%** of input, integer `floor(gross × 45 / 10000)` |
| LiFi EVM service | Currently **25 bps**, quote-derived (`feeSplit.lifiFee`); not paid to Smart Wallet |
| Combined LiFi swap | Currently **0.70%** service/platform; gas and DEX impact are separate |
| When it is charged | Encoded in the **same** source-chain transaction. If that transaction is rejected or fails, neither the swap nor the Smart Wallet fee completes. |
| How many times | **Once per signed payload.** Displayed Smart Wallet fee must match the encoded atomic fee or execute fails closed. |
| External DEX / Send / History view | **0** Smart Wallet platform fee |

EVM LiFi routes that cannot independently prove the 45/85 bps treasury credit fail closed. There is no post-trade collect and no unpaid-fee obligation.

---

## 5. Quote coordinator (EVM)

`SmartWalletEvmSwap.quoteSequential` / `quoteEvm`.

### 5.1 Hard rules

| Rule | Implementation |
|------|----------------|
| **No parallel fan-out** | `QUOTE_MAX_PARALLEL = 0`. Providers run **one after another**. |
| **Stale cancel** | `quoteSeq` — a newer keystroke abandons in-flight work. |
| **In-flight join** | Same quote key shares one promise (`INFLIGHT_BY_KEY`). |
| **Short caches** | Executable pack **8s**. No-route **4s**. |
| **LiFi cooldown** | On 429 / Retry-After: **20s** default, cap **120s**. Cooldown skips LiFi; next provider may still run. |
| **Official routers only** | V2 / V3 addresses remain recorded. Direct 0x / V2 / V3 execute paths stay **disabled**. |
| **No API keys** | Extension talks only to the staging LiFi Worker. The Worker holds `LIFI_API_KEY`. Never put that key in the extension or this repo. |

### 5.2 Provider order (same chain)

```text
1. LiFi          Staging Worker /v1/lifi/quote (+ routes only if needed)
                 ← live path for ETH / Polygon / Base / BSC / RH / Arb / OP / Avalanche
2. 0x            listed — runtime DISABLED (`no_backend` / no silent fallback)
3. Official V2   recorded — runtime DISABLED
4. Official V3   recorded — runtime DISABLED
```

**Arbitrum / Optimism / Avalanche Internal DEX (0.11.156–159):** LiFi is the working quote/execute coordinator. Live LiFi quotes were probed (ETH→USDC on 42161 and 10; AVAX→USDC on 43114). Do **not** treat the 0x chain allowlist as complete support. Token addresses are per-chain (see [CHAINS.md](./CHAINS.md)).

First **ok** pack wins. A LiFi 429 is not a hard fail of the whole quote. A cancelled / obsolete quote stops the walk.

### 5.3 Official V2 routers (verified)

| chainId | Router | Name |
|---------|--------|------|
| 1 | `0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D` | Uniswap V2 |
| 56 | `0x10ED43C718714eb63d5aA57B78B54704E256024E` | PancakeSwap V2 |
| 137 | `0xa5E0829CaCEd8fFDD4De3c43696c57F7D7A678ff` | QuickSwap V2 |
| 8453 | `0x4752ba5DBc23f44D87826276BF6Fd6b1C372aD24` | Uniswap V2 (Base) |

Wrapped natives are the canonical WETH / WBNB / WPOL / WETH-on-Base addresses. Selectors are the well-known `swapExactETHForTokens` / `swapExactTokensForETH` / `swapExactTokensForTokens` plus fee-on-transfer variants.

### 5.4 Official V3 (BNB only)

PancakeSwap V3 periphery:

- Router `0x1b81D678ffb9C0263b24A97847620C99d213eB14`
- Quoter `0xB048Bbc1Ee6b733FFfCFb9e9CeF7375518e25997`
- Fee tiers tried: **100 / 500 / 2500 / 10000**

Local ABI encode + `eth_call`. Not Infinity UR. Not the hosted Smart Router API.

### 5.5 Bounded BNB connector routing (0.11.99)

For thin BNB pairs the coordinator may search **connector** hops through a **fixed, on-chain-verified** set:

| id | Role |
|----|------|
| WBNB | wrap |
| USDT / USDC / FDUSD | stables |
| XAUt | verified connector |

Bounds (do not raise without a product reason):

| Cap | Value |
|-----|--------|
| Max hops | **3** |
| Max candidates | **14** |
| Max RPC in the search | **28** |
| Wall clock | **8s** |
| Pair cache | 5 min |
| Path-quote cache | 4s |

Addresses + decimals were verified on publicnode (2026-08-13). Symbols are display-only.

---

## 6. Solana path (Jupiter)

Same Swap panel, different engine.

| Step | Network |
|------|---------|
| Quote | `https://lite-api.jup.ag/swap/v1` (plus price / token search as needed) |
| Execute | Fresh quote + swap build + sign-once + `sendTransaction` |
| Confirm | Signature status / `getTransaction` — not an invented success |
| Platform fee | After success: convert (if needed) → USDC → treasury transfer |

**Jupiter fee stages** (0.11.150), durable key `smart_wallet_jup_fee_stage_v1`:

```text
NOT_STARTED
    → CONVERT_SUBMITTED
        → CONVERT_CONFIRMED
            → TRANSFER_SUBMITTED
                → PAID
         ↘ UNCERTAIN   (timeout / missing result — do not start a second convert)
```

Rules:

- `CONVERT_SUBMITTED` / `UNCERTAIN` **cannot** start another convert or 2-step.
- Survives popup restart.
- Failed **main** swap still collects **no** fee.
- 45 bps unchanged.

---

## 7. Preflight caches (`swap-preflight.js`)

Short-lived, privacy-safe (data **fingerprints**, not addresses / calldata in diagnostics):

| Cache | TTL |
|-------|-----|
| `eth_estimateGas` | 8s |
| Fee data (base / tip / gasPrice) | 4s |
| Simulation | 8s |
| Allowance | 8s |
| SOL rent / ATA | 10s / 15s |
| Fee-convert quote | 12s |
| Native balance | 3s |

Hits join in-flight promises so Confirm does not fire the same `estimateGas` twice. Critical send/sign/broadcast is **never** blocked by the request budget (see [LOADS.md](./LOADS.md)).

---

## 8. Execute — sign once, broadcast that raw

### 8.1 Software EVM (0.11.150)

`signAndBroadcastEvmSoftwareOnce`:

1. Build **one** immutable intent (to / value / data / nonce / gas / fees / chainId).
2. Sign **once**. Keep the local hash.
3. Sequential failover **rebroadcasts that raw only**.
4. Deterministic reject (`PENDING_BALANCE_RESERVED`, insufficient confirmed, nonce, revert) **stops**.
5. Timeout / coalesce may resubmit the **same** raw — never a newly signed replacement for the same click.

### 8.2 Ledger EVM

Same sign-once contract, plus:

- Cache key includes **gasLimit and fee fields**.
- Before reuse, **decode** the signed raw. Refuse if chain / from / to / value / nonce / gas / fees differ (0.11.139).
- Blind signing required for heavy router calldata.
- Permit2 (external DEX, not Internal DEX) has its own hashed-typed-data fallback — not used to build internal quotes.

### 8.3 Allowance

If the from-token is ERC-20, the wallet checks allowance against the **quote’s spender** (LiFi / 0x / official router). Approve is a **separate** signed tx, waited to confirm on L2/BSC/RH before the swap is sent. Wrong spender is `WRONG_ROUTER_SPENDER`, not a silent send.

---

## 9. Confirm truth and the later-receipt fee (0.11.152 / 0.11.155)

This is the Internal DEX fee-integrity contract.

### 9.1 States

**Main swap** (`mainSwap`):

| State | Meaning |
|-------|---------|
| `SUBMITTED` | Network returned a hash. Receipt not in (or not yet read). |
| `CONFIRMED` | Receipt **status 1**. |
| `FAILED` | Receipt **status 0**. |
| `UNCERTAIN` | Probe window expired (15 min). No resubmit. |

**Platform fee** (`platformFee`):

| State | Meaning |
|-------|---------|
| `NOT_DUE` | Default. Also the only legal state after `FAILED`. |
| `DUE` | Main swap confirmed; collect **once** when the wallet is unlocked. |
| `IN_FLIGHT` | Collect tx signed / in flight. |
| `PAID` | Fee transfer recorded. |
| `UNCERTAIN` | Collect or confirm timed out. Do not start a second convert. |

### 9.2 Hash without receipt

`executeSwap` does **not** set `confirmed` on a hash alone. History writes **pending**. Fee is **not** collected.

It persists `smart_wallet_swap_await_confirm_v1` via `queueSwapAwaitConfirm`:

- `mainSwap = SUBMITTED`
- `platformFee = NOT_DUE`
- `nextAt = now + 4s`
- `expiresAt = now + 15 minutes`
- identity = anti-double-charge key or the hash

### 9.3 Service-worker probe

Alarm `sw-swap-await-confirm`. Function `probeSwapAwaitReceiptsBg`:

| Allowed | Forbidden |
|---------|-----------|
| `eth_getTransactionReceipt(hash)` through the sequential gateway | `eth_sendRawTransaction` |
| Backoff 4s → 8s → 15s → 30s → 60s → 120s | Sign / rebroadcast / fee convert |
| Persist new state | Collect while locked |

On status **1**: `CONFIRMED` + `DUE` (unless already `PAID` / `IN_FLIGHT` / `UNCERTAIN`).  
On status **0**: `FAILED` + `NOT_DUE`.  
On expiry: both sides `UNCERTAIN`.

When `DUE`, SW messages the UI `smart-wallet-run-swap-await`. **Collection still requires an unlocked signer.** Residual / later-receipt recovery does not run while locked (honest limitation, not a silent charge).

Unlock calls `tryCompleteDueSwapFees`.

---

## 10. Outcome state machine (`swap-outcome.js`)

User-visible swap life:

```text
IDLE → QUOTING → QUOTE_READY
                 → AWAITING_RECONFIRMATION   (quote moved enough to ask again)
                 → CHECKING_ALLOWANCE → APPROVING → APPROVAL_CONFIRMED
                 → AWAITING_SIGNATURE → SIGNED → BROADCASTING
                 → CONFIRMING → CONFIRMED
                 → SETTLING_PLATFORM_FEE → COMPLETED
                         ↘ FEE_PENDING_RECOVERY
         BROADCAST_UNCERTAIN / REVERTED / FAILED_PRE_BROADCAST / CANCELLED
```

Illegal transitions are rejected. Correction caps stop “retry until it works” loops (dead-route loop fixed 0.11.102; fallback reconfirm 0.11.100–0.11.101).

Primary rate-limit with a working fallback is a **success with a code** (`PRIMARY_RATE_LIMITED_FALLBACK_OK`), not a red error.

---

## 11. Error handoff

Internal DEX does **not** have a second user-copy system.

| Layer | Who names it |
|-------|----------------|
| Quote / route / allowance / sim revert | `swap-outcome.js` + revert decoder |
| Funds / pending reserved / RPC / Ledger / broadcast | [Error System](./ERROR-SYSTEM.md) |
| Confirm / fee due | later-receipt machine above |

A LiFi “no route” is `NO_ROUTE`, not “not enough BNB.”  
A Geth queued-cost reject mid-swap is `PENDING_BALANCE_RESERVED`, not “raise slippage.”  
A hash with no receipt is **Submitted — confirming…**, not **Swap failed**.

---

## 12. Residual fee recovery (0.11.148 / 0.11.153)

After a **confirmed** internal swap or bridge, leftover USDC that should have gone to treasury is persisted **before** settlement.

| Rule | Behavior |
|------|----------|
| Timing | Fire-and-forget. **Not** awaited on the success toast. |
| Coalesce | Per-residual `nextAt` (**4s**). `FEE_RESIDUAL_INFLIGHT` dedupes runners. |
| Skip | Identities already `PAID` or `UNCERTAIN`. |
| Popup closed | SW alarm `sw-fee-residual` wakes the UI. UI must be **unlocked** to sign. |
| Math | Still 45 / 85 bps. No extra fee. |

---

## 13. What Internal DEX never does

- Parallel quote fan-out of free RPCs or aggregator HTTPS.
- Unofficial / unverified routers or pasted calldata as a “route.”
- Invent `0x1` or mark confirmed without receipt status 1.
- Charge 45 bps on a failed, reverted, or still-pending main swap.
- Start a second Jupiter convert from `CONVERT_SUBMITTED` / `UNCERTAIN`.
- Re-sign a changed amount from a Ledger cache hit.
- Auto-switch the wallet’s active chain to follow a quote.
- Touch external-DEX inject, WalletConnect, or Send fee policy.

---

## 14. Tests (docs names only)

| Script | Covers |
|--------|--------|
| `tools/test-evm-swap-providers.js` | Sequential coordinator, routers, cooldown |
| `tools/test-evm-swap-execute.js` / `execute-after-quote` / `live-path` | Execute after a good quote |
| `tools/test-evm-swap-sim-fallback.js` | Pre-sign sim revert → next provider |
| `tools/test-evm-connector-routing.js` | BNB connector bounds |
| `tools/test-evm-dead-route-loop.js` | No infinite fallback |
| `tools/test-swap-outcome.js` / `test-swap-preflight.js` / `test-swap-quote-lifecycle.js` | State + caches |
| `tools/test-swap-await-confirm-fee.js` | Later-receipt fee due; SW probe does not sign |
| `tools/test-batch1-tx-fee-dapp.js` | Jupiter stages + software sign-once |
| `tools/test-fee-reject-does-not-block.js` | Rejected fee ≠ blocked external DEX |
| `tools/test-swap-holdings-truth.js` / `test-swap-flip.js` | Panel truth |

---

## 15. Related product history (Internal DEX only)

| Version | Change |
|---------|--------|
| 0.11.35–36 | Confirm lag ≠ failed; chain-specific tx identity |
| 0.11.41 | EVM platform fee **after** main-swap confirm (`requirePaid`) |
| 0.11.91–102 | Shared EVM reliability, sequential fallback, BNB connectors, dead-route stop, Confirm Updated Route |
| 0.11.132 | Rejected fee does not block external DEX / bridge UI |
| 0.11.150 | Software sign-once + Jupiter fee stages |
| 0.11.152 | `confirmed` only on receipt status 1 |
| 0.11.153 | Per-residual recovery + SW `sw-fee-residual` |
| 0.11.155 | Later-receipt `SUBMITTED` → probe → `DUE` / `NOT_DUE` |
| 0.11.156–159 | Arbitrum / Optimism / Avalanche on the shared LiFi EVM path + USDC seeds |

---

## Related docs

| File | Role |
|------|------|
| [ERROR-SYSTEM.md](./ERROR-SYSTEM.md) | Inspect / classify / present / Logs |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Whole-wallet build |
| [CHAINS.md](./CHAINS.md) | Arb / OP / Avalanche tokens and RPC |
| [LOADS.md](./LOADS.md) | Quote / probe / fee HTTPS + RPC counts |
| [BUGS-AND-FIXES.md](./BUGS-AND-FIXES.md) | Fixed vs open |
| [Chrome-extension-store-for-reviewers/FEE-DISCLOSURE.md](./Chrome-extension-store-for-reviewers/FEE-DISCLOSURE.md) | Store fee disclosure |

---

*Not financial advice. Cryptocurrency involves risk of loss. Quotes are not guarantees of execution price.*
