# Smart Wallet — Chrome Web Store readiness

**Audience:** Operator / submitter (gap analysis — not marketing copy)  
**Product:** Smart Wallet (Chrome / Opera MV3) — authenticity tree **Smart Wallet R** (`React-Wallet` @ 0.11.698). Production folder `Smart-Wallet-0.6.55` is a separate tree — do not conflate.  
**Readiness snapshot:** **0.11.698** (stamp **w41**; reassessed **2026-09-13** after hard pass)  
**Previous snapshot stamps:** 0.11.698 refresh 2026-09-12; body dated 2026-08-20; earlier 0.11.164  
**Date:** 2026-09-13  

This file compares **current infrastructure** against **Chrome Web Store (CWS) submission + approval expectations**.

**Related public docs:** [HOST-PERMISSIONS.md](./Chrome-extension-store-for-reviewers/HOST-PERMISSIONS.md) · [CONTACTS.md](./Chrome-extension-store-for-reviewers/CONTACTS.md) · [FEE-DISCLOSURE.md](./Chrome-extension-store-for-reviewers/FEE-DISCLOSURE.md) · [PRIVACY-POLICY.md](./Chrome-extension-store-for-reviewers/PRIVACY-POLICY.md) · [MESSAGING.md](./MESSAGING.md) · [STORE-LISTING.txt](./STORE-LISTING.txt) · [CHAINS.md](./CHAINS.md) · live `ACTIVE.md` / `HOST-PERMISSIONS-STAGE.md` in the unpacked tree

---

## 1. Executive scorecard

| Area | CWS expectation | Status (0.11.698 @ 2026-09-13) | Risk if submit now |
|------|-----------------|--------------------------------|--------------------|
| Manifest V3 | Required | **Ready** — MV3, service worker, CSP `script-src 'self' 'wasm-unsafe-eval'; object-src 'self'; worker-src 'self'` | Low |
| Single purpose | Clear, narrow purpose | **Ready** — crypto wallet; optional address messaging is in-product mail, not a second product | Low |
| Package hygiene | No secrets, no dev junk | **Partial** — packager + secret scan OK; **current** `dist-store` zip is `dirty=True` (~68 dirty files, `-AllowDirty`). Sep 12 freeze zip was verifier-clean — **do not submit today’s dirty zip** | Med |
| Owner / admin UI | Not in customer zip | **Ready** — store strip removes Owner tools, Broadcast, device-auth, admin-token hooks | Low |
| Privacy policy URL | Public, accurate, linked | **Ready** — [PRIVACY-POLICY.md](./Chrome-extension-store-for-reviewers/PRIVACY-POLICY.md) (messaging included) | Low |
| In-product privacy | Accessible disclosure | **Ready** — `privacy.html` + Settings + messaging consent | Low |
| Permission justification | Every permission justified | **Ready** in [STORE-LISTING.txt](./STORE-LISTING.txt) — must paste on form; update for **optional** `clipboardRead` | Med (process) |
| Host permissions | Minimum necessary | **Improved** — **367** named declared hosts (Pass 1 shrink + Workers); optional `https://*/*` / `wss://*/*` / `http://*/*`; no localhost declared | Med (optional `*` + list size) |
| Content scripts | Scope matches purpose | **Strong** — **118** inject apexes / **~236** matches; not `<all_urls>` | Low–Med |
| Remote code / eval | Forbidden | **Ready** — no remote script; `wasm-unsafe-eval` only for vault WASM | Low |
| Screenshots / listing | Required assets + copy | **Copy ready; screenshots still missing** | **Blocker** |
| Support contact | Required on form | **Ready on GitHub** — must still paste on CWS form | Low–Med |
| Security of keys | No exfiltration | **Stronger** — Rust/WASM vault; `rustJsVaultForbidden` always true (no plaintext popup `SESSION_VAULT_PASSWORD`); MessageChannel-only wallet events; failsafe fallbacks; optional clipboardRead + paste-hijack gate (2026-09-13 hard pass) | Low |
| Fees transparency | Honest disclosure | **Ready** — 45 / 85 bps atomic; LiFi EVM currently +25 bps | Low–Med |
| Messaging honesty | Buttons must match live servers | **Improved vs Aug write-up** — [MESSAGING.md](./MESSAGING.md) says production mail-privacy live on Worker **0.2.29**. Still **operator-smoke** Delete for me / Block / Report / delete-all on production before submit | Low–Med (smoke, not architecture) |
| Managed RPC | Must not be silently on | **Blocked** — not live default; normal use is **public RPC** | Low |
| Listing name vs tree | Matches product | **Check** — unpacked manifest name is **Smart Wallet R**; Store listing should use the customer name (**Smart Wallet**) | Med |
| Trademarks / icons | Own or licensed | License PDF in repo | Med |
| Developer account | One-time fee, 2FA | Operator | Process |
| Cold-path resilience | Functional package | **Usable** — smoke-ready unpacked; `app.js` still ~71k on CORE (panel re-defer parked). Not a CWS reject by itself | Low–Med (reviewer UX) |

**Overall:** Architecture remains **Store-shaped**. Hard pass (2026-09-13) improved security / packaging honesty; it did **not** clear listing screenshots. Messaging server-privacy is documented live on **0.2.29** (reassess — was a hard blocker when production was 0.2.12). **Do not submit** the current dirty AllowDirty zip; rebuild from a clean freeze after VERIFY OK.

### Rough readiness (0.11.698 @ 2026-09-13)

| Lens | Estimate |
|------|----------|
| **Package + docs readiness** | **~90–93%** (was ~88–91% on 09-12 stamp) |
| **Likely first-pass approval** | **~55–65%** after screenshots + clean zip + messaging smoke (wallets often need 1–3 rounds) |
| **Usable as Load unpacked** | **~90%+** (operator smoke confirmed post hard pass) |

### What improved since the 2026-09-12 stamp / 0.11.164 era

- Hard pass (2026-09-13): failsafe DOM fallbacks; inject apexes covered by hosts; MessageChannel-only wallet events; password read gating; CSP `worker-src 'self'`; `clipboardRead` **optional**; store Include fixed for panel views + `send-asset-ui.js`; `package-report.txt` SHA aligned to zip `91623374…`.
- Host Pass 1: declared hosts **~396 → 367** (+ Worker hosts); staged plan in `HOST-PERMISSIONS-STAGE.md`.
- Messaging: production mail-privacy documented on Worker **0.2.29** ([MESSAGING.md](./MESSAGING.md)) — clears the old “stuck on 0.2.12” architecture blocker pending live smoke.
- Optional wallet Messaging / Inbox with on-device consent; Store ZIP strips Owner/Broadcast/device-auth.
- Packager refuses `STAGING-NOT-FOR-STORE`; no localhost in declared hosts.

### What still gates approval

- **Store screenshots are still missing** (P0).
- **Clean freeze zip:** commit/approve baseline → `verify-extension.ps1` VERIFY OK → `build-store-package.ps1` **without** `-AllowDirty` → archive that zip. Today’s zip is dirty.
- **Messaging production smoke** of server delete/block/report/delete-all against **0.2.29** (doc says live; CWS honesty still needs operator proof).
- Optional `https://*/*` / `wss://*/*` / `http://*/*` dashboard sentence (Custom RPC / relays only; inject stays allowlisted).
- Align dashboard listing name with **Smart Wallet** (not “Smart Wallet R” experiment chrome) unless intentional.
- Crypto wallets remain high-scrutiny.

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
| Messaging | Production RPC gateway mail host | Personal mail only after on-device consent; privacy APIs on Worker **0.2.29** per MESSAGING.md |
| Logs | None off-device | Local `chrome.storage` only |
| Managed RPC | **Off** | Public RPC mode |

### 3.3 Store package pipeline

```text
build-store-package.ps1
  → refuse STAGING-NOT-FOR-STORE / STORE-FREEZE mismatch
  → (canonical dist-store only) tools/verify-extension.ps1
  → copy allowlisted files (Include list)
  → strip owner HTML (Broadcast, Owner tools, device-auth)
  → secret scan
  → dist-store/Smart-Wallet-chrome-store.zip + package-report.txt
```

**Must stay out of the zip:** `.env`, `tools/`, recovery artifacts, operator notes, staging copies, owner mutation UI, admin tokens, secrets.

**Live package note (2026-09-13):** zip SHA `916233744a7fb2357c27bbc0faa22f19cc3017c7a24049b50d7a275e302bddf3`, `dirty=True`, built with `-AllowDirty` (verify skipped via non-canonical OutDir then copied). **Submission candidate = next clean rebuild only.**

---

## 4. Requirement vs build

### 4.1 Manifest

| Requirement | Status |
|-------------|--------|
| Manifest V3 | Yes |
| Service worker | `background.js` |
| CSP extension pages | `script-src 'self' 'wasm-unsafe-eval'; object-src 'self'; worker-src 'self'` |
| Icons 16/32/48/128 | Present |
| Version | **0.11.698** (stamp **w41**) — `STORE-FREEZE.txt` / `manifest.json` are authoritative. Git branch names like `fix/opera-resume-transaction-0.11.529` are **not** the product version — never publish 0.11.529 as live. |
| Minimum Chrome | 116 |
| Extension name in manifest | **Smart Wallet R** (experiment) — confirm Store listing title |

### 4.2 Permissions

| Permission | Why | Reviewer view |
|------------|-----|----------------|
| `storage` | Wallet, vault, settings, Logs, messaging consent | Standard |
| `clipboardWrite` | Copy address | Justify |
| `clipboardRead` | **Optional** — paste-hijack clear / read only when granted in-flow | Better than install-time; update STORE-LISTING paste |
| `offscreen` | Local signing | Acceptable |
| `scripting` | Allowlisted dApp inject only | Tie to dApp connect |
| `alarms` | Auto-lock | Good security story |
| `tabs` | Focus wallet for approve / Ledger | Justify |
| Declared `host_permissions` (**363**) | Named RPCs, LiFi/Jupiter/CoinGecko, explorers, WC, production Workers / mail | Keep list honest; Pass 2+ still planned |
| Optional `https://*/*`, `wss://*/*`, `http://*/*` | Custom RPC / extra relays only, requested at runtime | Explain; not install-time |
| localhost | **Not in declared hosts** | Good |

Content scripts: **118** inject apexes (not every website). Optional `*` does **not** inject everywhere.

### 4.3 Privacy

| Requirement | Status |
|-------------|--------|
| Public privacy policy | Includes messaging |
| In-extension privacy | `privacy.html` |
| Messaging consent | Separate on-device key; never sent to the Worker |
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

Recommended screenshot set: Home, Send, Swap **with fee line**, Settings (Privacy / Contact / Fees), optional History, optional **Inbox** (no message bodies in the capture). Capture from the **clean store zip**, not a dirty unpacked profile.

---

## 5. Messaging and Store (important)

Customer Messaging **belongs in the Store ZIP**. Owner moderation and Broadcast **do not**.

| Control | Store ZIP | Production Worker privacy |
|---------|-----------|---------------------------|
| Inbox / Sent / Compose / Announcements (read) | Yes | Send/pull/announcements on production |
| Delete conversation, Delete selected, Clear inbox | Yes | **Local device only** |
| Delete for me | Yes | Server `/v1/mail/delete-for-me` — **0.2.29** per MESSAGING.md; **smoke before submit** |
| Request deletion of my server messages | Yes | Server `/v1/mail/delete-all` — same |
| Block / Unblock / Blocked addresses | Yes | Server block routes — same |
| Report message | Yes | Server report — same |
| Broadcast announcements | **Stripped** | Unpacked owner only |
| Owner tools / Message reports | **Stripped** | Unpacked owner only |

**Stop condition:** do not submit until operator smoke proves those server actions against production **0.2.29** (or temporarily ship UI that cannot claim server deletion). Local-only deletes remain safe.

Full control map: [MESSAGING.md](./MESSAGING.md).

---

## 6. Gap list

### P0 — Blockers before first submit

1. **Screenshots** from the **clean store zip**.  
2. **Clean rebuild:** freeze → VERIFY OK → `build-store-package.ps1` (no `-AllowDirty`) → archive; do not upload today’s dirty zip.  
3. **Messaging production smoke** (Delete for me / Block / Report / delete-all) on Worker **0.2.29**.  
4. **Paste support contact** on the CWS form — [CONTACTS.md](./Chrome-extension-store-for-reviewers/CONTACTS.md).  
5. **One paragraph on optional `*` hosts** — Custom RPC / relays only; inject stays allowlisted.

### P1 — High value

6. Permission justification paste from STORE-LISTING.txt (include optional `clipboardRead`).  
7. Fee + privacy URLs on the form.  
8. Confirm icon trademark / license.  
9. Align listing copy with networks, 45/85 bps, optional Helius, optional Inbox; listing title **Smart Wallet**.  
10. Live-click smoke of the **zip**: create/lock, Solana swap, one EVM send, dApp connect, Messaging consent + local delete + server actions.  
11. Host Pass 2 when review heat on list size matters.

### P2 — After v1

12. Further cold-path splits / panel re-defer (reliability, not a CWS checkbox).  
13. Full password envelope-only redesign (already refuse-hold on Rust).  
14. Public status page.

---

## 7. “Would we pass today?”

| Scenario | Likely outcome |
|----------|----------------|
| Clean zip + screenshots + host essay + messaging smoke on 0.2.29 | **Possible**, often with clarification on optional hosts / crypto risk |
| Submit without screenshots | **Rejected / incomplete** |
| Submit today’s dirty AllowDirty zip | **Don’t** — process / trust smell; rebuild clean |
| Submit Messaging server UI that 404s | **Honesty / functionality fail** |
| Submit a STAGING-NOT-FOR-STORE tree | Packager **refuses**; do not bypass |
| Submit with owner tools / Broadcast / admin token | **Reject** — strip already prevents this if the packager is used |
| Submit secrets or recovery tools | **Reject / trust damage** |

Plan for **1–3 review iterations**.

---

## 8. Recommended submit path

1. Smoke-accept production mail-privacy on **0.2.29** (Delete for me / Block / Report / delete-all). Do not enable Managed RPC.  
2. Land/commit an approved freeze baseline (clean tree).  
3. Confirm privacy + fee + contact + [MESSAGING.md](./MESSAGING.md) + STORE-LISTING (`clipboardRead` optional) match the zip.  
4. `tools\verify-extension.ps1` → **VERIFY OK**.  
5. `build-store-package.ps1` (**no** `-AllowDirty`) → archive that zip; confirm `package-report.txt` SHA matches.  
6. Capture screenshots from the zip loaded unpacked.  
7. Smoke the zip: create wallet, lock/unlock, small swap, one send, dApp connect, Inbox consent, local + server messaging actions.  
8. Dashboard: zip, listing title **Smart Wallet**, permission justifications, privacy URL, support email, fee disclosure URL.  
9. Review answers: non-custodial, local keys, allowlisted inject, declared hosts, optional `*` only for Custom RPC, Messaging optional and not E2E encrypted.

---

## 9. One-line verdict

**Smart Wallet (0.11.698) is an MV3 wallet with a verify-gated, owner-stripped store pipeline, hardened hard-pass security (2026-09-13), 367 named hosts, optional clipboardRead, and documented production mail-privacy on Worker 0.2.29. Chrome Web Store submit is still gated by missing screenshots and a clean freeze zip (+ messaging smoke) — not by missing basic extension architecture.**

**~90%+ ready to assemble a complete package after those P0 items; ~55–65% chance of first-pass approve without a revision request (normal for wallets). Load-unpacked daily use: yes. Submit today: no.**

---

*Not financial advice. Not a guarantee of Chrome Web Store approval. Re-check live CWS policies before each submission. Do not publish extension source. Reassessed from local tree `C:\Users\levyr\Desktop\React-Wallet` on 2026-09-13 (not from GitHub).*