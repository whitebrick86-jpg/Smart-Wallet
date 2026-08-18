# Smart Wallet — Chrome Web Store readiness

**Audience:** Operator / submitter (gap analysis — not marketing copy)  
**Product:** Smart Wallet (Chrome / Opera MV3)  
**Readiness snapshot:** **0.11.164** (last full CWS gap analysis)  
**Live product:** **0.11.257** — rebuild the store zip at freeze; do not upload an older `dist-store` file. See [PRODUCT.md](./PRODUCT.md).  
**Date:** 2026-08-13  

This file compares **current infrastructure** against **Chrome Web Store (CWS) submission + approval expectations**.

**Related public docs:** [HOST-PERMISSIONS.md](./Chrome-extension-store-for-reviewers/HOST-PERMISSIONS.md) · [CONTACTS.md](./Chrome-extension-store-for-reviewers/CONTACTS.md) · [FEE-DISCLOSURE.md](./Chrome-extension-store-for-reviewers/FEE-DISCLOSURE.md) · [PRIVACY-POLICY.md](./Chrome-extension-store-for-reviewers/PRIVACY-POLICY.md) · [STORE-LISTING.txt](./STORE-LISTING.txt) · [CHAINS.md](./CHAINS.md) · [ERROR-SYSTEM.md](./ERROR-SYSTEM.md) · [allow-list/](./allow-list/)

---

## 1. Executive scorecard

| Area | CWS expectation | Our status (0.11.164) | Risk if submit now |
|------|-----------------|------------------------|--------------------|
| Manifest V3 | Required | **Ready** — MV3, service worker, no remote code | Low |
| Single purpose | Clear, narrow purpose | **Ready** — crypto wallet only | Low |
| Package hygiene | No secrets, no dev junk | **Ready** — `build-store-package.ps1` + `verify-extension.ps1` gate | Low |
| Privacy policy URL | Public, accurate, linked | **Ready** — GitHub [PRIVACY-POLICY.md](./Chrome-extension-store-for-reviewers/PRIVACY-POLICY.md) | Low |
| In-product privacy | Accessible disclosure | **Ready** — in-extension privacy page / Settings links | Low |
| Permission justification | Every permission justified | **Ready in [STORE-LISTING.txt](./STORE-LISTING.txt)** — must paste in dashboard | Med (process) |
| Host permissions | Minimum necessary | **Improved** — declared RPC/API/explorer/WC hosts; `https://*/*` + `wss://*/*` are **optional** (user-selected Advanced/custom RPC). Localhost is not in the production package. | **Med–High** (wallet + optional `*`) |
| Content scripts | Scope matches purpose | **Strong** — **105** apex inject hosts only ([allow-list/](./allow-list/)); includes Arb / OP / Avalanche dApps | Low–Med |
| Remote code / eval | Forbidden | **Ready** — CSP `script-src 'self'` | Low |
| Screenshots / listing | Required assets + copy | **Copy ready; screenshots TBD** | **Blocker** |
| Support contact | Required on form | **Ready on GitHub** — [CONTACTS.md](./Chrome-extension-store-for-reviewers/CONTACTS.md) | Low–Med (paste on CWS form too) |
| Security of keys | No exfiltration | **Strong** — always-encrypted local vault / Ledger / JIT secrets | Low |
| Remote hosting of logic | No | **Ready** — all logic in package | Low |
| Fees transparency | Honest disclosure | **Ready** — Smart Wallet 45/85 bps atomic; LiFi EVM currently +25 bps; [FEE-DISCLOSURE.md](./Chrome-extension-store-for-reviewers/FEE-DISCLOSURE.md) | Low–Med |
| Error honesty | No fake “failed” / no raw dumps | **Ready** — [ERROR-SYSTEM.md](./ERROR-SYSTEM.md) + local Logs (no upload) | Low (helps review) |
| Trademarks / icons | Own or licensed | **Check** — logo license PDF in repo; chain icons resized to Base-class PNGs | Med |
| Developer account | One-time fee, 2FA | **Operator** — not in repo | Process |
| Review narrative | Consistent story | **Docs-only public repo** vs closed source OK | Low |

**Overall:** Technically close to a **store zip**. **Approval risk is still dominated by missing listing screenshots + high-scrutiny wallet review + optional broad hosts / localhost.** Required `https://*/*` is no longer the install-time default (it is optional). That is better than 0.11.0, not a free pass.

### Rough readiness (0.11.164)

| Lens | Estimate |
|------|----------|
| **Package + docs readiness** | **~84–88%** |
| **Likely first-pass approval** | **~55–65%** (wallets often need 1–3 revision rounds) |
| **Usable as Load unpacked / private zip** | **~90%+** |

**What improved since the 0.11.0 readiness note**

- Declared `host_permissions` are **named** RPC / API / explorer / WC hosts (including official Arb / OP / Avalanche). Broad `https://*/*` + `wss://*/*` moved to **`optional_host_permissions`**.
- Networks: **11** (Solana, Ethereum, Bitcoin, Polygon, Sui, Robinhood, Base, BNB, **Arbitrum One, Optimism, Avalanche C-Chain**). Shared EVM Send / LiFi Swap / LiFi Bridge — see [CHAINS.md](./CHAINS.md).
- Error System + privacy-safe **Logs** (local `chrome.storage` only, FIFO 100). No crash reporter, no upload.
- Inject allow-list **105** apex hosts (was smaller); still **not** `<all_urls>`.
- Chain icons for Arb / OP / AVAX shrunk to 128×128 (package size down; last rebuilt store zip **0.11.160** ~2.38 MB). Live product is **0.11.164** — **rebuild the zip at freeze**.
- Verify suite is the store-package gate (`VERIFY OK — 0.11.164`).

**What barely moved approval odds**

- **Store screenshots are still missing.**
- Crypto wallets remain high-scrutiny.
- `optional_host_permissions` of `https://*/*` / `wss://*/*` need the dashboard sentence that they are optional, user-selected Advanced RPC only. Localhost is not in the production package.

---

## 2. What CWS cares about (condensed)

Chrome's review looks for:

1. **Single purpose** — description, UI, and permissions all match.  
2. **Permissions** — each one necessary; broad host access is scrutinized heavily.  
3. **Privacy** — policy URL works; data practices match code.  
4. **No remote code** — no downloading/executing JS from the network.  
5. **User safety** — crypto wallets are high-scrutiny; misleading fee/copy or phishing-adjacent UX is fatal.  
6. **Complete listing** — icons, screenshots, privacy, support, category.  
7. **Functional package** — zip loads, popup works, no crash on open.

Official framing (re-check live policies before submit): [Chrome Web Store Program Policies](https://developer.chrome.com/docs/webstore/program-policies/).

---

## 3. Current infrastructure map (what we ship)

### 3.1 Runtime surfaces

| Surface | Files | Role |
|---------|-------|------|
| Popup UI | `popup.html`, `app.js`, `styles.css`, `live-feeds.js` | Home, send, swap, bridge, history, settings, Logs |
| Full page | `index.html` | WC host, Ledger, approve deep links |
| Service worker | `background.js` | dApp bridge, auto-lock, session, inject, residual/receipt alarms |
| Page inject | `page-boot.js`, `injected.js`, `content-script.js`, `inject-allowlist.js` | Wallet Standard + EIP-1193 on **allowlisted** hosts |
| Offscreen | `offscreen.html`, `offscreen-sign.js` | Local sign helpers |
| Errors / DEX | `evm-error-classify.js`, `tx-error-present.js`, `swap-outcome.js`, `sw-diag-log.js` | Classify / present / Logs |
| Config | `config.js`, `chain-registry.js` | Non-secret public config + chain registry |

### 3.2 Data / network

| Path | Destination class | Notes |
|------|-------------------|--------|
| Balances / send | Public chain RPCs (sequential multi-endpoint) | HTTPS; official Arb / OP / Avalanche C-Chain included |
| Prices | Jupiter lite, CoinGecko (incl. `avalanche-2`), Binance ticker WS | Idle-first; **one** ticker for the **active** native |
| Swap | Jupiter (Solana), **LiFi** (EVM) | Smart Wallet **0.45%** atomic; LiFi EVM currently **0.70%** combined with LI.FI 0.25% |
| Bridge | LiFi (EVM source) | Smart Wallet **0.85%** atomic; currently **1.10%** combined with LI.FI 0.25% |
| History | Public RPC / Blockscout-class explorers or optional Helius | User-supplied key only |
| WC | Reown / WalletConnect relays | User Project ID |
| Logs | **None** | Local `chrome.storage.local` only |
| Fee collection | Atomic in the source-chain swap/bridge transaction (not post-trade) | See [FEE-DISCLOSURE.md](./Chrome-extension-store-for-reviewers/FEE-DISCLOSURE.md) |

### 3.3 Store package pipeline

```text
build-store-package.ps1
  → tools/verify-extension.ps1   (syntax, version pins, full node suite)
  → dist-store/package/          (allowlisted files only)
  → dist-store/Smart-Wallet-chrome-store.zip
```

**Typically included:** manifest, UI, core JS, icons, fonts, lib, privacy.html, live-feeds.js.  
**Excluded (must stay out):** `.env`, `tools/`, recovery tools, private operator notes, Python scripts, secrets, session notes.

**Operator:** last packaged zip was **0.11.160**. Live code is **0.11.164**. Run `build-store-package.ps1` again at submit freeze.

---

## 4. Side-by-side: requirement vs our build

### 4.1 Manifest & technical compliance

| Requirement | Our infrastructure | Gap / action |
|-------------|-------------------|--------------|
| Manifest V3 | Yes (`manifest_version: 3`) | None |
| Service worker background | `background.js` | None |
| CSP restricts extension pages | `script-src 'self'; object-src 'self'` | Keep it that way |
| No `eval` / remote script for logic | Logic is local; price/RPC are data | Maintain |
| Icons 16/32/48/128 | Present under `icons/` | Chain logos now 128×128 class |
| Version string | **0.11.164** | Freeze + rebuild zip before upload |
| Minimum Chrome | 116 | OK |

### 4.2 Permissions

| Permission | Why we use it | CWS reviewer view | Gap |
|------------|---------------|-------------------|-----|
| `storage` | Wallet state, vault, settings, Logs | Standard | Justify |
| `clipboardWrite` / `clipboardRead` | Copy address; paste guard | Sensitive | Justify (in STORE-LISTING) |
| `offscreen` | Signing helpers | Acceptable for wallets | Justify |
| `scripting` | Inject / tab provider | Sensitive | Tie to dApp connect only |
| `alarms` | Auto-lock + bounded residual/receipt jobs | Good security story | OK |
| `tabs` | Focus wallet for approve / Ledger | Common | Justify |
| `hid` | Ledger USB | Strong justification | OK |
| **Declared `host_permissions`** | Named public RPCs, LiFi/Jupiter/CoinGecko, explorers (incl. arbiscan / optimistic.etherscan / snowtrace / official Arb/OP/AVAX RPC), WC relays | Reviewer can read the list | Keep list honest |
| **`optional_host_permissions`: `https://*/*`, `wss://*/*`** | Custom RPC, extra WC/relays, rare failover hosts | Still sensitive — but **not** forced at install | Explain in dashboard + [HOST-PERMISSIONS.md](./Chrome-extension-store-for-reviewers/HOST-PERMISSIONS.md) |
| localhost / 127.0.0.1 | Local dApps | Often questioned on store builds | **Optional strip for store zip** |

**Important nuance:** Content scripts are **allowlisted** (DEX/DeFi hosts only — 105 apex names). Global **optional** hosts enable **extension ↔ network** when the user (or a feature) needs an endpoint not frozen in the declared list. They do **not** inject on every website.

**Mitigation options:**

1. Submit as-is: declared hosts + optional `*` + strong justification (expect a question).  
2. Strip localhost from the **store** manifest only.  
3. Do not grant optional `*` unless the user adds a custom RPC / WC path.

### 4.3 Privacy & disclosure

| Requirement | Our status | Gap |
|-------------|------------|-----|
| Public privacy policy URL | [PRIVACY-POLICY.md](./Chrome-extension-store-for-reviewers/PRIVACY-POLICY.md) | Keep in sync with code |
| Contact method | [CONTACTS.md](./Chrome-extension-store-for-reviewers/CONTACTS.md) | Paste same emails on CWS dashboard |
| In-extension privacy | privacy page + Settings | OK |
| Third parties listed | Helius, Jupiter, LiFi, CoinGecko, Binance ticker, WC, public RPCs | Refresh when features change |
| No seed to our servers | True by design | Maintain |
| Logs | Local only; redacted; no upload | Mention — helps vs “telemetry” |
| Fees | [FEE-DISCLOSURE.md](./Chrome-extension-store-for-reviewers/FEE-DISCLOSURE.md) + in-app Swap/Bridge lines | Keep 45 / 85 bps aligned |

### 4.4 Listing & dashboard assets

| Asset | Required | Our status |
|-------|----------|------------|
| Short description | Yes | Ready (`STORE-LISTING.txt` / manifest) |
| Detailed description | Yes | Ready — mention multi-chain including Arb / OP / Avalanche |
| Category | Yes | Operator picks (follow current CWS list) |
| Screenshots 1280×800 or 640×400 | Yes | **Not prepared in repo** → **blocker** |
| Small/large promo tiles | Optional/required by size | Prepare if prompted |
| Privacy policy field | Yes | URL ready |
| Homepage | Yes | This GitHub docs repo |
| Support URL / email | Yes | CONTACTS.md — **put on form** |
| Single purpose declaration | Yes | Text in STORE-LISTING |

### 4.5 Security / product behavior (reviewer-sensitive)

| Topic | Our infrastructure | Approval note |
|-------|-------------------|---------------|
| Non-custodial | Local keys / Ledger | Align all copy with this |
| Password vault | AES-GCM + PBKDF2 (~650k); always at rest | Good |
| Auto-lock | 15m–3h; session generation so Lock wins | Good story |
| Platform fees | 0.45% / 0.85% Smart Wallet + current LI.FI 0.25% on EVM (0.70% / 1.10% combined) | Disclose separately from gas/relayer |
| dApp inject | Allowlist only (105 hosts) | Helps vs "scrapes every site" |
| Error System | Inspect → classify → present → stamp | No raw RPC dumps in UI |
| Recovery / dev tools | Must **not** be in store zip | Keep out forever |
| Dev server / `.env` | Not in zip | Keep out |

### 4.6 Operational packaging checklist

| Step | Ready? |
|------|--------|
| `build-store-package.ps1` produces zip | Yes |
| Zip root has `manifest.json` | Enforced by script |
| Zip has in-extension privacy page | Enforced |
| No `.env` / pem / secrets | Script strips |
| `VERIFY OK` on freeze version | Yes on **0.11.164** in the live tree |
| Load unpacked smoke of **zip contents** | Operator must do (rebuild zip first) |
| Developer dashboard account + $5 one-time | Operator |
| 2FA on Google account | Operator |

---

## 5. Gap list prioritized for approval

### P0 — Blockers before first submit

1. **Screenshots** (min 1–5): Home, Send, Swap (**fee line visible**), Settings (Privacy / Contact / Fee disclosure / Logs), optional History. Capture from the **store zip**, not a dirty dev profile.  
2. **Rebuild + smoke-test the store zip** at **0.11.164** (or the freeze you actually upload).  
3. **Paste support contact on CWS form** — [CONTACTS.md](./Chrome-extension-store-for-reviewers/CONTACTS.md).  
4. **One paragraph on optional `https://*/*`** — custom RPC / WC only; inject stays allowlisted.

### P1 — High value before / during review

5. **Permission justification paste** — use `STORE-LISTING.txt` verbatim; mention optional vs declared hosts.  
6. **Fee + privacy URLs** on form.  
7. **Optional:** strip localhost from the production store manifest.  
8. **Confirm icon trademark / license.**  
9. **Align listing** with live behavior: 11 networks, LiFi EVM, Jupiter Solana, 45/85 bps, optional Helius, no custody.  
10. **Live-click smoke** on freeze: create/lock, Solana swap, one EVM send (Base or Arb), dApp connect (jup.ag / app.uniswap.org). Arb/OP/AVAX quotes were probed; in-wallet Send/Swap **execute** was not click-certified.

### P2 — Nice to have / post-v1

11. Separate "dev" vs "store" manifest generation.  
12. Automated zip audit (file list + forbidden patterns).  
13. Public status page or support email automation.

---

## 6. "Would we pass today?" scenarios

| Scenario | Likely outcome |
|----------|----------------|
| Submit rebuilt zip + screenshots + host essay (declared + optional `*`) | **Possible**, often with a **clarification** on optional hosts / crypto risk |
| Submit without screenshots | **Rejected / incomplete** |
| Submit 0.11.160 zip while listing says 0.11.164 | **Confusion / bounce** — rebuild |
| Submit with secrets or recovery tools in zip | **Reject / trust damage** |
| Submit with vague purpose ("productivity tools") | **Reject** |
| Drop optional `*` and localhost without testing custom RPC / local dApps | **Easier review**, some user paths break |

Wallets are **not** rubber-stamped. Plan for **1–3 review iterations**.

---

## 7. Comparison summary table

| Infrastructure we have | Maps to CWS need | Readiness |
|------------------------|------------------|-----------|
| MV3 + CSP + local JS | Technical compliance | Ready |
| Allowlisted content scripts (105) | Safer inject story | Ready / strong |
| Declared RPC/API hosts + optional `*` | Network flexibility without forced `*` at install | Better than 0.11.0; still explain |
| localhost in declared hosts | Local dApps | Strip for store if possible |
| Privacy.md + privacy.html | Privacy disclosure | Ready |
| CONTACTS.md (emails) | Support contact | Ready (paste on form) |
| FEE-DISCLOSURE.md + Settings | Fee honesty | Ready |
| ERROR-SYSTEM + local Logs | Honest failures, no phone-home | Ready / helpful |
| STORE-LISTING.txt | Dashboard copy | Ready (refresh chain list) |
| build-store-package.ps1 + verify | Clean artifact | Ready — rebuild at 0.11.164 |
| Vault + auto-lock + Ledger | Security narrative | Ready |
| 11 networks, shared EVM | Product completeness | Ready to describe |
| Public docs repo (no source) | Transparency without open-sourcing keys | OK |
| Screenshots / promo | Listing completeness | **Missing** |

---

## 8. Recommended submit path (operator)

1. Freeze a version (**0.11.164** or bump to **1.0.0** for first store).  
2. Confirm privacy + fee + contact pages match live UI.  
3. Run `tools\verify-extension.ps1` → **VERIFY OK**.  
4. Run `build-store-package.ps1` → archive that zip (do not upload an older `dist-store` file).  
5. Capture screenshots from the **zip package** loaded unpacked.  
6. Smoke the zip: create wallet, lock/unlock, small Solana swap, one EVM send, dApp connect (e.g. jup.ag), Ledger optional.  
7. Dashboard: upload zip; paste listing + permission justifications; privacy URL; support email from [CONTACTS.md](./Chrome-extension-store-for-reviewers/CONTACTS.md); fee disclosure URL.  
8. Answer review questions with the same story: non-custodial wallet, local keys, **allowlisted inject**, **declared** RPC/API hosts, **optional** `*` only for custom RPC / extra relays (see [HOST-PERMISSIONS.md](./Chrome-extension-store-for-reviewers/HOST-PERMISSIONS.md)).

---

## 9. One-line verdict

**Smart Wallet is a real MV3 wallet with a verify-gated store zip, public privacy/contact/fee docs, an Error System that does not upload logs, and eleven networks on a shared EVM path. Chrome Web Store approval is still gated by missing screenshots, standard high-risk wallet review, and how clearly we explain optional broad hosts — not by missing basic extension architecture.**

**~86% ready to submit a complete package once a 0.11.164 zip is rebuilt, screenshots exist, and that zip is smoke-tested; ~55–65% chance of first-pass approve without a revision request (normal for wallets).**

---

*Not financial advice. Not a guarantee of Chrome Web Store approval. Re-check live CWS policies before each submission.*
