# Chrome extension store (for reviewers)

**Product:** Smart Wallet — Chrome / Opera Manifest V3 non-custodial multi-chain wallet  
**Audience:** Chrome Web Store reviewers, security reviewers, compliance readers  
**Docs snapshot:** 0.11.253  

This folder is a **single place** with the documents most relevant to Chrome Web Store review.  
You do not need to search the rest of the repository for privacy, fees, host permissions, or contacts.

**Documentation-only repository** — extension source code is **not** published here.  
The shipped package is the store zip built from the operator package pipeline.

---

## Start here (recommended order)

| # | Document | Why it matters for review |
|---|----------|---------------------------|
| 1 | **[HOST-PERMISSIONS.md](./HOST-PERMISSIONS.md)** | **Justification** for global `https://*/*` / `wss://*/*` vs **allowlisted** content-script inject |
| 2 | **[PRIVACY-POLICY.md](./PRIVACY-POLICY.md)** | Full privacy policy (store privacy policy URL target) |
| 3 | **[privacy.html](./privacy.html)** | In-extension privacy summary (same product story, shorter) |
| 4 | **[FEE-DISCLOSURE.md](./FEE-DISCLOSURE.md)** | Smart Wallet 0.45%/0.85% plus current LI.FI 0.25% on EVM (0.70%/1.10% combined) |
| 5 | **[CONTACTS.md](./CONTACTS.md)** | Developer + support emails |

---

## Quick facts for reviewers

| Topic | Summary |
|-------|---------|
| **Purpose** | Non-custodial browser crypto wallet only (create/import, balances, send, swap, bridge, dApp connect, optional Ledger) |
| **Custody** | No Smart Wallet cloud custody of seeds/keys |
| **Inject scope** | Content scripts / wallet provider on an **allowlist** of DEX/DeFi hosts — not every website |
| **Network scope** | Broad `host_permissions` so the **extension** can call RPCs, price APIs, swap/bridge APIs, WC relays, and optional user-pasted RPC (see host-permissions doc) |
| **Remote code** | Product logic ships in the extension package; extension pages CSP `script-src 'self'` |
| **Fees** | Smart Wallet 0.45% Jupiter / 0.45%+current LI.FI 0.25% LiFi swap / 0.85%+0.25% LiFi bridge; atomic with the source tx; not on Send / external dApps |
| **Support** | See [CONTACTS.md](./CONTACTS.md) |

---

## Suggested privacy policy URL (store form)

```text
https://github.com/Greenwolf30/Smart-Wallet/blob/main/Chrome-extension-store-for-reviewers/PRIVACY-POLICY.md
```

**Homepage / docs root:**  
https://github.com/Greenwolf30/Smart-Wallet  

**This reviewer pack:**  
https://github.com/Greenwolf30/Smart-Wallet/tree/main/Chrome-extension-store-for-reviewers  

---

## Contact

| Role | Address |
|------|---------|
| Developer | Whitebrick86@gmail.com |
| Support | smartwallethelp@outlook.com |

Full page: [CONTACTS.md](./CONTACTS.md)

---

## Files in this folder

| File | Role |
|------|------|
| [README.md](./README.md) | This index |
| [HOST-PERMISSIONS.md](./HOST-PERMISSIONS.md) | Host permission **justification** |
| [PRIVACY-POLICY.md](./PRIVACY-POLICY.md) | Full privacy policy |
| [privacy.html](./privacy.html) | In-product privacy summary |
| [FEE-DISCLOSURE.md](./FEE-DISCLOSURE.md) | Fee disclosure |
| [CONTACTS.md](./CONTACTS.md) | Contact emails |

These are the **canonical** public privacy, fee, host-permission, and contact documents for Smart Wallet.\r\n\r\n**Not included here (operator only — for the submitter, not reviewers):**

| File | Role |
|------|------|
| [STORE-LISTING.txt](../STORE-LISTING.txt) | Dashboard paste kit (listing text + permission form answers) |
| [CHROME-WEB-STORE-READINESS.md](../CHROME-WEB-STORE-READINESS.md) | Internal gap analysis (screenshots, packaging checklist) |

---

*Not financial advice. Cryptocurrency involves risk of loss. This pack does not guarantee Chrome Web Store approval.*
