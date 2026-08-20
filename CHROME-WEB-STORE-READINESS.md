# Smart Wallet — Chrome Web Store readiness

**Audience:** Operator / submitter (gap analysis — not marketing copy)  
**Product:** Smart Wallet (Chrome / Opera MV3)  
**Readiness snapshot:** **0.11.388** (this reassessment)  
**Previous snapshot:** 0.11.164  
**Date:** 2026-08-20  

This file compares **current infrastructure** against **Chrome Web Store (CWS) submission + approval expectations**.

**Related public docs:** [HOST-PERMISSIONS.md](./Chrome-extension-store-for-reviewers/HOST-PERMISSIONS.md) · [CONTACTS.md](./Chrome-extension-store-for-reviewers/CONTACTS.md) · [FEE-DISCLOSURE.md](./Chrome-extension-store-for-reviewers/FEE-DISCLOSURE.md) · [PRIVACY-POLICY.md](./Chrome-extension-store-for-reviewers/PRIVACY-POLICY.md) · [MESSAGING.md](./MESSAGING.md) · [STORE-LISTING.txt](./STORE-LISTING.txt) · [CHAINS.md](./CHAINS.md)

---

## 1. Executive scorecard

| Area | CWS expectation | Status (0.11.388) | Risk if submit now |
|------|-----------------|-------------------|--------------------|
| Manifest V3 | Required | **Ready** — MV3, service worker, CSP `script-src 'self'` | Low |
| Single purpose | Clear, narrow purpose | **Ready** — crypto wallet; optional address messaging is in-product mail, not a second product | Low |
| Package hygiene | No secrets, no dev junk | **Ready** — verify gate + allowlisted zip + secret scan; **STAGING-NOT-FOR-STORE** trees are refused | Low |
| Owner / admin UI | Not in customer zip | **Ready** — store strip removes Owner tools, Broadcast, device-auth, admin-token hooks | Low |
| Privacy policy URL | Public, accurate, linked | **Ready** — [PRIVACY-POLICY.md](./Chrome-extension-store-for-reviewers/PRIVACY-POLICY.md) updated **2026-08-20** (messaging included) | Low |
| In-product privacy | Accessible disclosure | **Ready** — `privacy.html` + Settings links + pre-use messaging consent | Low |
| Permission justification | Every permission justified | **Ready** in [STORE-LISTING.txt](./STORE-LISTING.txt) | Med (process: paste on form) |
| Host permissions | Minimum necessary | **Improved** — named declared hosts; `https://*/*` + `wss://*/*` **optional**; **localhost / 127.0.0.1 not declared** | Med (optional `*` still scrutinized) |
| Content scripts | Scope matches purpose | **Strong** — **105** apex inject hosts; not `<all_urls>` | Low–Med |
| Remote code / eval | Forbidden | **Ready** | Low |
| Screenshots / listing | Required assets + copy | **Copy ready; screenshots still missing** | **Blocker** |
| Support contact | Required on form | **Ready on GitHub** — must still paste on CWS form | Low–Med |
| Security of keys | No exfiltration | **Strong** — encrypted vault / Ledger / JIT secrets | Low |
| Fees transparency | Honest disclosure | **Ready** — 45 / 85 bps atomic; LiFi EVM currently +25 bps | Low–Med |
| Messaging honesty | Buttons must match live servers | **Store UI is customer-safe; production Worker mail-privacy is not deployed yet** | **Blocker for submit** |
| Managed RPC | Must not be silently on | **Off / Public RPC** | Low |
| Trademarks / icons | Own or licensed | License PDF in repo | Med |
| Developer account | One-time fee, 2FA | Operator | Process |

**Overall:** The **package architecture is Store-shaped**. First submit is still blocked by **missing listing screenshots**, the need to **rebuild and smoke the freeze zip**, and **production Worker mail-privacy not being live**. Optional `https://*/*` remains a review question, not a free pass. Crypto wallets remain high-scrutiny.

### Rough readiness (0.11.388)

| Lens | Estimate |
|------|----------|
| **Package + docs readiness** | **~88–91%** |
| **Likely first-pass approval** | **~55–65%** once P0 blockers are cleared (wallets often need 1–3 rounds) |
| **Usable as Load unpacked** | **~90%+** |

### What improved since 0.11.164

- Live unpacked product is **0.11.388** (was 0.11.257 in older README copy).
- Optional wallet **Messaging / Inbox** with a separate on-device consent notice (**2026-08-20**). See [MESSAGING.md](./MESSAGING.md).
- Store ZIP **strips** Owner tools, Broadcast announcements composer, owner device-auth script, and admin-token hooks. Customer Inbox, Sent, Compose, Announcements (read), Block, Report, Delete for me, and deletion request remain.
- Packager **refuses** any tree labeled `STAGING-NOT-FOR-STORE`.
- Declared hosts no longer include **localhost / 127.0.0.1**.
- Privacy policy and in-extension privacy page cover messaging: not end-to-end encrypted, Delete for me vs server deletion request, block, report.
- Secret scan + store-owner-separation tests are part of VERIFY.
- Canonical unpacked mail host stays **production**; staging is a separate non-store copy only.

### What still gates approval

- **Store screenshots are still missing.**
- **Do not upload a customer zip that exposes Messaging deletion/block/report against production Worker 0.2.12.** Those APIs were proven on staging 0.2.15. Production Worker, production KV, and production Durable Objects were not changed in the staging-acceptance pass.
- Crypto wallets remain high-scrutiny.
- Optional `https://*/*` / `wss://*/*` still need the dashboard sentence: not granted at install; requested only for a user-pasted Custom RPC origin.

---

## 2. What CWS cares about (condensed)

1. **Single purpose** — description, UI, and permissions match.  
2. **Permissions** — each one necessary; broad host access is scrutinized.  
3. **Privacy** — policy URL works; data practices match code.  
4. **No remote code** — no downloading or executing JS from the network.  
5. **User safety** — misleading fee copy, phishing-adjacent UX, or buttons that claim deletion they cannot perform is fatal.  
6. **Complete listing** — icons, screenshots, privacy, support, category.  
7. **Functional package** — zip loads, popup works, no crash on open.

Official framing: [Chrome Web Store Program Policies](https://developer.chrome.com/docs/webstore/program-policies/).

---

## 3. Current infrastructure map

### 3.1 Runtime surfaces (customer Store ZIP)

| Surface | Role |
|---------|------|
| Popup / full page | Home, send, swap, bridge, history, settings, Logs, **Inbox / Messaging** |
| Service worker | dApp bridge, auto-lock, inject, residual/receipt alarms |
| Page inject | Wallet Standard + EIP-1193 on **allowlisted** hosts only |
| Offscreen | Local sign helpers |
| Privacy / consent | `privacy.html`, privacy consent, **messaging consent** (separate key) |

**Stripped from Store ZIP (unpacked-only):** owner device enrollment, Managed RPC mode switches, Broadcast composer, Owner tools / message-report administration, `rpc-device-auth.js`.

### 3.2 Data / network

| Path | Destination | Notes |
|------|-------------|-------|
| Balances / send | Public chain RPCs | Sequential multi-endpoint |
| Prices | Jupiter lite, CoinGecko, Binance ticker | Idle-first |
| Swap | Jupiter (Solana), LiFi (EVM) via **production** LiFi proxy | Atomic 0.45% Smart Wallet |
| Bridge | LiFi (EVM source) via production LiFi proxy | Atomic 0.85% Smart Wallet |
| History | Public RPC / explorers or optional user Helius key | |
| WC | Reown / WalletConnect | User Project ID |
| Messaging | Production RPC gateway mail host | Personal mail only after on-device consent |
| Logs | None off-device | Local `chrome.storage` only |
| Managed RPC | **Off** | Public RPC mode |

### 3.3 Store package pipeline

```text
build-store-package.ps1
  → refuse STAGING-NOT-FOR-STORE
  → tools/verify-extension.ps1
  → copy allowlisted files
  → strip owner HTML (Broadcast, Owner tools, device-auth)
  → secret scan
  → dist-store/Smart-Wallet-chrome-store.zip
```

**Must stay out of the zip:** `.env`, `tools/`, recovery artifacts, operator notes, staging copies, owner mutation UI, admin tokens, secrets.

**Do not** create or upload the freeze zip until the operator authorizes it. An older `dist-store` file is not the 0.11.388 freeze.

---

## 4. Requirement vs build

### 4.1 Manifest

| Requirement | Status |
|-------------|--------|
| Manifest V3 | Yes |
| Service worker | `background.js` |
| CSP extension pages | `script-src 'self'; object-src 'self'` |
| Icons 16/32/48/128 | Present |
| Version | **0.11.388** — freeze + rebuild zip at submit |
| Minimum Chrome | 116 |

### 4.2 Permissions

| Permission | Why | Reviewer view |
|------------|-----|----------------|
| `storage` | Wallet, vault, settings, Logs, messaging consent | Standard |
| `clipboardWrite` / `clipboardRead` | Copy address; paste guard | Justify |
| `offscreen` | Local signing | Acceptable |
| `scripting` | Allowlisted dApp inject only | Tie to dApp connect |
| `alarms` | Auto-lock | Good security story |
| `tabs` | Focus wallet for approve / Ledger | Justify |
| `hid` | Ledger USB | Strong justification |
| Declared `host_permissions` | Named RPCs, LiFi/Jupiter/CoinGecko, explorers, WC, **production mail gateway** | Keep list honest |
| Optional `https://*/*`, `wss://*/*` | Custom RPC / extra relays only, requested at runtime | Explain; not install-time |
| localhost | **Not in declared hosts** | Better than 0.11.164 |

Content scripts inject only on **105** apex DEX/DeFi hosts. Optional `*` does **not** inject on every website.

### 4.3 Privacy

| Requirement | Status |
|-------------|--------|
| Public privacy policy | 2026-08-20, includes messaging |
| In-extension privacy | `privacy.html` |
| Messaging consent | Separate on-device key `smart_wallet_messaging_consent_v1`; never sent to the Worker |
| No seed to servers | True by design |
| Logs | Local only |
| Fees | 45 / 85 bps + current LiFi 0.25% on EVM |

### 4.4 Listing assets

| Asset | Status |
|-------|--------|
| Short / detailed description | Ready in STORE-LISTING.txt |
| Privacy / homepage / support URLs | Ready |
| Screenshots 1280×800 or 640×400 | **Missing — blocker** |
| Promo tiles | Prepare if prompted |

Recommended screenshot set: Home, Send, Swap **with fee line**, Settings (Privacy / Contact / Fees), optional History, optional **Inbox** (no message bodies in the capture).

---

## 5. Messaging and Store (important)

Customer Messaging **belongs in the Store ZIP**. Owner moderation and Broadcast **do not**.

| Control | Store ZIP | Needs production Worker privacy APIs |
|---------|-----------|--------------------------------------|
| Inbox / Sent / Compose / Announcements (read) | Yes | Send/pull/announcements already gated on production 0.2.12 flags |
| Delete conversation, Delete selected, Clear inbox | Yes | **Local device only** — safe without new Worker routes |
| Delete for me | Yes | **Server** `/v1/mail/delete-for-me` — staging-proven; **not production-deployed** |
| Request deletion of my server messages | Yes | **Server** `/v1/mail/delete-all` — staging-proven; **not production-deployed** |
| Block / Unblock / Blocked addresses | Yes | **Server** block routes — staging-proven; **not production-deployed** |
| Report message | Yes | **Server** report + coordinator — staging-proven; **not production-deployed** |
| Broadcast announcements | **Stripped** | Unpacked owner only |
| Owner tools / Message reports | **Stripped** | Unpacked owner only |

**Stop condition:** do not submit a customer Store ZIP that invites users to Delete for me, Block, Report, or Request deletion until production Worker mail-privacy is deployed and smoke-tested. Local-only delete conversation may ship earlier; mixed buttons that 404 against production 0.2.12 would fail review honesty.

Full control map: [MESSAGING.md](./MESSAGING.md).

---

## 6. Gap list

### P0 — Blockers before first submit

1. **Screenshots** from the **store zip**, not a dirty unpacked profile.  
2. **Rebuild + smoke** `build-store-package.ps1` at the freeze version (currently 0.11.388). Do not upload an older zip.  
3. **Production Worker mail-privacy** (or freeze a zip whose Messaging buttons cannot falsely claim server deletion). Production remains **0.2.12**. Staging is **0.2.15**. Report coordinator production binding is prepared and **not migrated**.  
4. **Paste support contact** on the CWS form — [CONTACTS.md](./Chrome-extension-store-for-reviewers/CONTACTS.md).  
5. **One paragraph on optional `https://*/*`** — Custom RPC only; inject stays allowlisted.

### P1 — High value

6. Permission justification paste from STORE-LISTING.txt.  
7. Fee + privacy URLs on the form.  
8. Confirm icon trademark / license.  
9. Align listing copy with 11 networks, 45/85 bps, optional Helius, optional Inbox.  
10. Live-click smoke of the **zip**: create/lock, Solana swap, one EVM send, dApp connect, Messaging consent + local delete (server actions only after production mail-privacy).

### P2 — After v1

11. Dedicated store vs unpacked manifest generation (already partly done via strip).  
12. Public status page.

---

## 7. “Would we pass today?”

| Scenario | Likely outcome |
|----------|----------------|
| Submit rebuilt zip + screenshots + host essay **after** production mail-privacy | **Possible**, often with a clarification on optional hosts / crypto risk |
| Submit without screenshots | **Rejected / incomplete** |
| Submit Messaging server-delete/block/report UI against production 0.2.12 | **Honesty / functionality fail** |
| Submit a STAGING-NOT-FOR-STORE tree | Packager **refuses**; do not bypass |
| Submit with owner tools / Broadcast / admin token | **Reject** — strip already prevents this if the packager is used |
| Submit secrets or recovery tools | **Reject / trust damage** |

Plan for **1–3 review iterations**.

---

## 8. Recommended submit path

1. Deploy and accept **production** mail-privacy (separate authorized pass). Do not enable Managed RPC.  
2. Freeze extension version.  
3. Confirm privacy + fee + contact + [MESSAGING.md](./MESSAGING.md) match the zip.  
4. `tools\verify-extension.ps1` → **VERIFY OK**.  
5. `build-store-package.ps1` → archive that zip.  
6. Capture screenshots from the zip loaded unpacked.  
7. Smoke: create wallet, lock/unlock, small swap, one send, dApp connect, Inbox consent, local delete, then server delete/block only if production APIs are live.  
8. Dashboard: zip, listing, permission justifications, privacy URL, support email, fee disclosure URL.  
9. Review answers: non-custodial, local keys, allowlisted inject, declared hosts, optional `*` only for Custom RPC, Messaging is optional and not E2E encrypted.

---

## 9. One-line verdict

**Smart Wallet is an MV3 wallet with a verify-gated, owner-stripped store pipeline, 2026-08-20 privacy/messaging disclosure, no localhost in declared hosts, and eleven networks. Chrome Web Store submit is still gated by missing screenshots, a freeze zip rebuild, and production Worker mail-privacy — not by missing basic extension architecture.**

**~90% ready to assemble a complete package after those P0 items; ~55–65% chance of first-pass approve without a revision request (normal for wallets).**

---

*Not financial advice. Not a guarantee of Chrome Web Store approval. Re-check live CWS policies before each submission. Do not publish extension source.*
