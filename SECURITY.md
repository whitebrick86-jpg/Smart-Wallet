# How Smart Wallet protects your keys

**Product:** Smart Wallet (Chrome / Opera MV3 browser wallet)  
**Live product:** **0.11.698** (stamp **w289**)  
**Last updated:** 2026-09-21  

Smart Wallet is **self-custodial**. You control your recovery phrase and keys. Smart Wallet does **not** hold your funds, cannot reset your wallet for you, and does **not** upload your seed phrase or private keys to our servers.

This page is a plain-language overview of how key protection works in the product — similar in spirit to the security guides other major wallets publish for users. It is **not** a third-party security audit and does **not** publish extension source code.

---

## What “self-custodial” means here

| You control | Smart Wallet does **not** |
|-------------|---------------------------|
| Your Secret Recovery Phrase (seed) | Recover a lost seed for you |
| Software keys on your device (encrypted) | Store your keys on our servers |
| Ledger keys on your hardware device | Import Ledger private keys into the extension |
| Every send, swap, bridge, or dApp signature you approve | Sign transactions without your approval |

If someone else gets your seed phrase, they can control the accounts derived from it. Treat it like cash and a password combined.

---

## Software wallets (seed / imported keys)

### Encrypted at rest

Your software-wallet secrets are stored **encrypted** in the browser extension’s local storage, along with public account metadata (addresses, settings). They are **not** kept as a plaintext seed or private keys on disk.

### Rust / WASM vault

Software-wallet create, import, unlock, backup, recovery, reveal, and routine signing are handled by Smart Wallet’s **Rust/WASM vault**. For day-to-day signing, key material is used **inside** that vault boundary; what leaves for a normal send or dApp signature is a **signature or signed transaction**, not your raw private key.

While the wallet is unlocked, the interface keeps a **vault session** — not your password, seed phrase, or private keys sitting in the page as plaintext.

### Password ON vs Password OFF

| Mode | What it means |
|------|----------------|
| **Password ON** | Your password protects the encrypted vault. After lock (or when the wallet asks), you unlock with that password. |
| **Password OFF** | The vault is still encrypted using a **device-wrap** flow so the seed is not stored as plaintext. There is no routine password prompt, but protection depends more on the security of this browser profile and device. |

A strong password plus a locked device is stronger than password-off device wrap against someone who fully compromises the browser profile.

### Lock and auto-lock

Manual lock and auto-lock end the live vault session so routine signing cannot continue until you unlock again. Wrong-password lockout is separate from the auto-lock timer you choose.

### Revealing a seed or private key

Viewing your seed or a chain private key is an **explicit** action. While that reveal screen is open, treat the secret as exposed to the page and anyone who can see your screen. Close and clear that view when you are done. Prefer writing the seed offline; never paste it into a website or chat.

---

## Hardware wallets (Ledger)

Ledger account **private keys stay on the Ledger device**. Smart Wallet only stores public addresses and derivation information needed to talk to the device.

Today, Ledger connect / signing in Smart Wallet covers **Solana** and **EVM** networks (Ethereum, Polygon, Base, BNB Smart Chain, Arbitrum, Optimism, Avalanche C-Chain, Robinhood Chain, Sonic, and related EVM flows after Link EVM).

**Bitcoin** and **Sui** are available as **software** wallet chains in Smart Wallet. They are **not** supported on Ledger accounts yet. Use a software / seed wallet for those chains for now; fuller hardware support is planned later. See [DOCUMENTATION.txt](./DOCUMENTATION.txt) (Ledger section) and [HOW-TO-MULTIPLE-LEDGER-WALLETS.txt](./HOW-TO-MULTIPLE-LEDGER-WALLETS.txt).

You approve sensitive Ledger operations **on the device**.

---

## What other parts of the wallet see

Balances, prices, quotes, history, messaging, and dApp discovery are built to work from **public** information and your approvals. They are **not** intentionally given your private keys.

dApp connect still requires you to approve connections and signatures. A secure vault cannot make an **approved** malicious transaction safe — always read what you are signing.

---

## Practical habits (same advice majors give)

1. **Never share** your seed phrase or private keys with anyone — including people claiming to be support.  
2. Store your seed **offline** (paper or metal), not in screenshots, email, or cloud notes.  
3. Prefer a **strong password** (Password ON) on shared or unlocked machines.  
4. For larger balances, prefer **Ledger** where supported (Solana + EVM today).  
5. Keep your browser and OS updated; avoid unknown extensions on the same profile.  
6. Review dApp approvals and revoke token spending you no longer need (see [external-dex-permissions.md](./external-dex-permissions.md) for how external DEX approvals behave).

---

## Privacy and store review

- In-extension privacy: **Settings → Privacy**, and the reviewer pack under [Chrome-extension-store-for-reviewers/](./Chrome-extension-store-for-reviewers/).  
- Full user guide: [DOCUMENTATION.txt](./DOCUMENTATION.txt).  
- Product overview (versions / fees): [PRODUCT.md](./PRODUCT.md).

---

## Bottom line

Smart Wallet keeps software secrets **encrypted on your device**, uses a **Rust/WASM vault** for software custody and signing, and keeps **Ledger keys on the hardware**. We cannot recover a lost seed. You stay in control — and responsible — for backups and for every transaction you approve.

*Documentation only. Extension source is not published in this repository.*
