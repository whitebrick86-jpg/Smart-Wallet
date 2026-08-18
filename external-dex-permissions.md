# External DEX permissions (ERC-20 approval audit)

**Product:** Smart Wallet  
**Scope:** External dApp DEX approvals on EVM (Uniswap, PancakeSwap, 1inch, LiFi, etc.)  
**Status:** Investigation / architecture notes — documents current behavior and recommendations.  
**Chains audited:** Ethereum, Base, Polygon, BSC, Robinhood  
**Wallets audited:** Software and Ledger  

---

# External EVM DEX ERC-20 approval audit

**Status: investigation only — no code changes required for this document.**

---

## 1. Current approval behavior

### Two separate systems

| Path | Who controls amount | Sign type | Post-swap revoke? |
|------|---------------------|-----------|-------------------|
| **Internal** Smart Wallet Swap (LiFi) | Wallet (`ensureEvmTokenAllowance`) | Classic ERC-20 `approve` | **Yes (software only)** |
| **External** dApp (Uniswap / PCS / 1inch) | **The dApp** | Classic `eth_sendTransaction` approve **and/or** EIP-712 **Permit2** | **No** |

External approvals are **passthrough**: Smart Wallet signs and broadcasts what the site builds. It does **not** rewrite approve amounts for external DEX.

---

### Internal path (for contrast — already implemented)

| Wallet type | Approve amount | After successful swap |
|-------------|----------------|------------------------|
| **Software** | Exact need **+ 1%** | `runPostSwapEvmRevoke` → `approve(spender, 0)` for that trade’s router/spender only |
| **Ledger** | **MaxUint256** (unlimited, once) | **No auto-revoke** (would need another Nano confirm) |

Shared helpers: `ensureEvmTokenAllowance`, `revokeEvmTokenAllowance`, `revokeEvmApprovalsAfterInternalSwap`, `runPostSwapEvmRevoke`.  
**Permit2 address is explicitly never revoked.**

---

### External path

```
dApp → inject → SW handleEvmProviderRequest
  ├─ eth_signTypedData_v4 (Permit2)  → UI approve → Ledger/software sign → signature to dApp
  └─ eth_sendTransaction (approve 0x095ea7b3 or swap)
        → UI confirm
        → Software: offscreen-sign (broadcast; short wait if approve)
        → Ledger: handleLedgerEvmSignRequest → signAndBroadcastEvmLedger
```

| Question | Answer |
|----------|--------|
| Exact vs unlimited? | **Whatever the dApp encodes.** Uniswap often does **unlimited ERC-20 approve → Permit2**, then **Permit2 permits** per trade. UI labels “Unlimited …” when amount ≥ MaxUint/2. |
| Residual after swap? | **Yes** for classic ERC-20 approve to a router/Permit2: residual stays until user/dApp changes it. Permit2 allowance to the Permit2 contract also typically stays. |
| Smart Wallet rewrites amount? | **No** on external path. |
| Auto-revoke after external swap? | **No** — only internal LiFi success path. |

---

## 2. Security risks

| Risk | Severity | Notes |
|------|----------|--------|
| Residual **unlimited** ERC-20 → router | Medium–high if dApp/router compromised | Site-chosen; wallet only signs |
| Residual ERC-20 → **Permit2** | Medium | Standard Uniswap model; Permit2 still needs a later signature for transfers |
| Auto-revoke **Permit2** | **High if done blindly** | Code already documents: breaks “Permit approval failed” on later sells |
| Auto-revoke **wrong spender** | High | Could brick other dApps sharing a router |
| **Ledger silent revoke** | Not possible without another device confirm | Current design correctly skips Ledger for silent revoke |

---

## 3. Gas / UX tradeoffs

| Policy | Extra txs | Ledger prompts | UX |
|--------|-----------|----------------|-----|
| Unlimited + never revoke (typical Uniswap) | 0 after first approve | 0 after first | Best convenience |
| Exact approve every trade | 1 approve + 1 swap each time | 2 prompts each sell | Safest, heavy |
| Exact + auto-revoke residual (internal software today) | +1 revoke after swap | N/A for software | Safer; extra gas ~approve cost |
| Auto-revoke on **Ledger** | +1 revoke tx | **+1 Nano confirm** | Users often reject; feels broken |
| Exact-only for external | Requires **rewriting** dApp calldata | Breaks sites that require max approve | **Not compatible** without site cooperation |

**Automatic revocation always costs another on-chain tx + gas** and, on Ledger, another device confirm. It cannot be free or silent on Ledger without keys in software.

---

## 4. Permit2 implications

| Topic | Finding |
|-------|---------|
| Address | `0x000000000022d473030f116ddee9f6b43ac78ba3` (shared across EVMs) |
| Flow | Often (1) ERC-20 `approve(Permit2, max)` once, (2) EIP-712 PermitSingle/Batch for each swap |
| Auto-revoke ERC-20 → Permit2 | **Unsafe** without product decision — next Uniswap sell needs re-approve; known failure mode in code comments |
| Auto-revoke Permit2 “allowance” to Universal Router | Different from ERC-20; not the same as `approve(0)` on token; **do not auto-touch** |
| Recommendation | **Exclude Permit2 and verifyingContract=Permit2 from any auto-revoke** (already true for internal) |

---

## 5. Chain differences (Ethereum / Base / Polygon / BSC / Robinhood)

| Area | Shared vs chain-specific |
|------|---------------------------|
| External approve passthrough | **Shared** (all EVM) |
| Internal exact vs Ledger unlimited | **Shared** policy |
| Internal post-swap revoke | **Shared**; software only |
| Fee floors for approve txs | Chain-specific floors, shared builder |
| Permit2 address | Same on all five chains |
| USDT-style zero-then-approve | Shared in `ensureEvmTokenAllowance` (internal) |

No separate external-approve implementation per chain. Robinhood only differs in RPC/fee floors, not allowance math.

---

## 6. Software vs Ledger (external)

| | Software | Ledger |
|--|----------|--------|
| External classic approve | Signs dApp amount as-is | Same |
| External Permit2 | EIP-712 sign only (no token move) | Same |
| Can silent-revoke after external swap? | Technically yes (keys in memory during session) | **No** without second Nano prompt |
| Internal post-swap revoke today | Yes | Skipped |

---

## 7. Shared helpers (change these, not chain forks)

| Helper | Role | External? |
|--------|------|-----------|
| `ensureEvmTokenAllowance` | Internal exact/unlimited approve | No |
| `revokeEvmTokenAllowance` | `approve(0)` software | Internal only today |
| `revokeEvmApprovalsAfterInternalSwap` | Scoped revoke + **skip Permit2** | Internal only |
| `runPostSwapEvmRevoke` | Post-internal success UI + revoke | Internal only |
| `handleEvmProviderRequest` / offscreen `ethSendTransaction` | External approve/swap | **External** — no amount rewrite |
| Typed-data path | Permit2 | **External** — signature only |

---

## 8. Should Smart Wallet auto-revoke after external DEX?

### Verdict: **Not by default as a silent automatic policy**

| Reason | Detail |
|--------|--------|
| Amount is dApp-controlled | Wallet cannot force “exact only” without mutating `eth_sendTransaction` data (breaks Uniswap/PCS/1inch expectations) |
| Permit2 is the dominant Uniswap path | Auto-revoking ERC-20→Permit2 is explicitly known to break sells |
| Ledger | Revoke = extra gas + **extra Ledger confirm**; cannot match software silent revoke |
| Success detection | External: wallet returns hash after broadcast; “swap success” is not always known before Uniswap finishes multi-step (approve then swap) |
| Gas | Users pay for a revoke they didn’t request |

### When it *could* be safe (product opt-in)

Only if **all** of:

1. Confirmed classic ERC-20 `approve` (not Permit2-only trade)  
2. Spender is **not** Permit2  
3. Software wallet (or user explicitly confirms Ledger revoke)  
4. Optional **user setting**: “Revoke residual allowances after dApp swaps”  
5. Scoped to **that** token + **that** spender observed in the approve tx  

---

## 9. Recommended architecture (proposal only)

### Principles

1. **Do not rewrite** external dApp `approve` calldata (keep site compatibility).  
2. **Never auto-revoke Permit2.**  
3. **Do not change** internal LiFi path (already exact + software revoke).  
4. Prefer **shared helpers**; no per-chain forks.  
5. Keep revoke/UI policy out of core sign/broadcast where possible.

### Optional phased design

| Phase | What | Safe? |
|-------|------|--------|
| **A – Observe only** | Decode external `approve` (token, spender, amount, unlimited?); log/history badge “Unlimited approval on Uniswap” | Yes |
| **B – Soft UX** | After dApp approve, offer optional “Revoke when done” or Settings toggle (software default off) | Yes |
| **C – Limited auto-revoke** | If toggle on + software + spender ≠ Permit2 + we saw matching approve earlier + user completed flow | Conditional |
| **D – Ledger auto-revoke** | **Not recommended** as automatic; optional “Revoke on Ledger?” after swap | Product call |
| **E – Force exact approve** | Mutate dApp approve amount | **Not recommended** |

### Minimal implementation sketch (if product later says yes)

```
// Shared, not chain-specific
// 1) On external eth_sendTransaction decode: if approve → remember {token, spender, amount, origin, chainId}
// 2) On later eth_sendTransaction success that looks like a swap involving token (optional heuristic)
//    OR explicit user "Revoke residual" action:
//    if software && spender !== PERMIT2 && residual > 0:
//       revokeEvmTokenAllowance(...)  // reuse existing helper
// 3) Never call this from Permit2 typed-data path
// 4) Ledger: only if user confirms a second prompt
```

Internal: leave `runPostSwapEvmRevoke` as-is.

---

## 10. Answers to numbered audit questions

1. **Exact or unlimited?** External: **dApp-chosen** (often unlimited for Uniswap→Permit2). Internal software: exact+1%; internal Ledger: unlimited.  
2. **Residual remains?** **Yes** for external classic approve; Permit2 token allowance also typically remains.  
3. **Can SW safely auto-revoke?** Only in **narrow software + non-Permit2 + explicit policy** cases; not silently for all external swaps.  
4. **Revoke = another tx + gas?** **Yes.** Ledger = **another device confirm.**  
5. **Interfere with Uniswap/PCS/1inch/LiFi?** Auto-revoking Permit2 or shared routers: **yes, breaks later sells.** Exact rewrite of approve: **high risk of site incompatibility.**  
6. **Permit2?** Signature ≠ ERC-20 move; **exclude from auto-revoke** (already documented for internal).  
7. **All chains same?** **Yes** for approval policy; only fees/RPC differ.  
8. **Shared helpers?** Extend `revokeEvmTokenAllowance` / policy layer — **do not** add Robinhood-only revoke logic.

---

## 11. What was intentionally left unchanged

This document does not require modifying working external Ledger broadcast, internal swap, RPC, gas/fee, or lifecycle paths.

---

## Bottom line

Smart Wallet **already** does residual revoke for **internal software** LiFi swaps and **correctly never** touches Permit2. For **external** DEX, auto-revoke-after-swap is **not** safe as a default: amounts are site-controlled, Uniswap is Permit2-heavy, and Ledger cannot silent-revoke. Prefer **optional, scoped, non-Permit2, software-first** revocation (or user-initiated) if stronger residual security is desired later—not a forced exact-approve rewrite for all dApps.

### Suggested next product steps (if implementing)

- **Phase A (observe/badge only)** — surface unlimited external approvals in UI/history  
- **Phase B (settings opt-in revoke)** — software users can enable residual revoke for non-Permit2 spenders  

---

*Document: external DEX permissions audit. Public docs repo: Greenwolf30/Smart-Wallet (documentation only, not extension source).*
