# Smart Wallet

**Current checkpoint:** **0.11.298** (Batch 5 verified). Worker is **1.6.0**. Store ZIP is built locally and is not published in this repository.

Non-custodial multi-chain **Chrome / Opera MV3** browser wallet.

- Solana, Ethereum (and EVM: Base, Polygon, BNB, Robinhood ETH), Bitcoin, Sui  
- Local keys always encrypted at rest Â· optional software password Â· Ledger (Solana + EVM)  
- dApp connect (Wallet Standard / EIP-1193) Â· WalletConnect Â· optional Helius History  

**Docs & privacy:** [github.com/whitebrick86-jpg/Smart-Wallet](https://github.com/whitebrick86-jpg/Smart-Wallet)

## Load unpacked (development)

1. Open `chrome://extensions` â†’ Developer mode  
2. **Load unpacked** â†’ select this folder (contains `manifest.json`)  
3. Pin Smart Wallet  

Do **not** remove the extension to update â€” overwrite files, then **Reload**.

## Chrome Web Store package

Build a clean zip (no `.env`, no dev scripts):

```powershell
cd path\to\this\folder
powershell -ExecutionPolicy Bypass -File .\build-store-package.ps1
```

Output: `dist-store\Smart-Wallet-chrome-store.zip`

### Store submission (you must submit)

1. [Chrome Web Store Developer Dashboard](https://chrome.google.com/webstore/devconsole)  
2. Upload the zip  
3. Privacy policy URL:  
   `https://github.com/whitebrick86-jpg/Smart-Wallet/blob/main/Chrome-extension-store-for-reviewers/PRIVACY-POLICY.md`  
4. Copy listing text from `STORE-LISTING.txt`  
5. Add screenshots Â· submit for review  

Listing copy: `STORE-LISTING.txt`. Full user docs: `DOCUMENTATION.txt` (fees: Â§18; dApp inject allowlist: Â§14.2).

## Optional: better Solana History

1. Free key: [dashboard.helius.dev](https://dashboard.helius.dev)  
2. Accounts â†’ **Advanced â€“ RPC** â†’ paste key â†’ Save  
3. History â†’ Solana â†’ Refresh  

Live balances / price pings do not require Helius.

## Privacy (in-extension)

Open **Settings â†’ Privacy** (in-extension privacy page / `Chrome-extension-store-for-reviewers/privacy.html`) or the full policy on GitHub.

## Local page mode (optional)

`serve.py` / `start.ps1` for a local tab + `.env` Helius â€” **not** required for the extension.

Never commit `.env` or personal API keys.
