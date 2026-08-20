# Chrome Web Store path

Operator checklist for packaging and submitting **Smart Wallet** to the Chrome Web Store.

Use the **live extension folder** on the machine (Load unpacked), not a stale copy. After code changes, rebuild the store zip before upload.

**Readiness snapshot:** **0.11.388**. Full gap analysis: [CHROME-WEB-STORE-READINESS.md](./CHROME-WEB-STORE-READINESS.md). Messaging map: [MESSAGING.md](./MESSAGING.md). Rebuild the zip at freeze — do not upload an older `dist-store` file. Do not submit until listing screenshots exist and production Worker mail-privacy matches the customer buttons.

---

## Steps

4. Run `tools\verify-extension.ps1` (should still be **VERIFY OK**).

5. Rebuild store zip: `build-store-package.ps1` (don’t upload stale `dist-store`).

6. Screenshots for the store listing.

7. Paste **STORE-LISTING** + privacy into the developer dashboard.

8. Submit when smoke + zip + screenshots are done.

---

## Related docs

| File | Role |
|------|------|
| [CHROME-WEB-STORE-READINESS.md](./CHROME-WEB-STORE-READINESS.md) | Gap analysis / readiness |
| [STORE-LISTING.txt](./STORE-LISTING.txt) | Listing copy to paste |
| [Chrome-extension-store-for-reviewers/](./Chrome-extension-store-for-reviewers/) | Privacy, fees, host permissions, contacts |
| [EXTENSION-README.md](./EXTENSION-README.md) | Install and package notes |
| [BUGS-AND-FIXES.md](./BUGS-AND-FIXES.md) | Open vs fixed issues |

---

*Not financial advice. Cryptocurrency involves risk of loss.*
