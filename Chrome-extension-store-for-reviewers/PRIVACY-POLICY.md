# Smart Wallet â€” Privacy Policy

**Last updated:** 2026-08-08  

## Who we are

Smart Wallet is a non-custodial browser extension that lets you hold and use cryptocurrency keys on your device and connect to decentralized applications (dApps).

**Documentation:** [https://github.com/whitebrick86-jpg/Smart-Wallet](https://github.com/whitebrick86-jpg/Smart-Wallet)

## Data we do not collect

- We do **not** operate a Smart Wallet cloud account that stores your seed phrase or private keys.  
- We do **not** intentionally transmit seed phrases or private keys to Smart Wallet servers (core wallet use does not require a Smart Wallet server account).  
- We do **not** sell your personal information for advertising.

## Data stored on your device

The extension stores wallet-related data **locally** in your browser profile, which may include:

- Public addresses and account labels  
- Encrypted vault data for software wallets (always encrypted at rest; user password or device wrap when password protection is off)  
- Settings (for example auto-lock duration, optional custom Solana RPC / Helius API key that **you** paste)  
- Local caches (for example recent history rows, price cache, WalletConnect session data)

You can remove this data by removing the extension or clearing the extension's storage for this browser profile.

## Network requests your browser makes

When you use Smart Wallet, **your browser** may contact third-party infrastructure you choose or that the app uses by default, for example:

| Destination | Purpose | Typical data |
|-------------|---------|----------------|
| Public Solana / EVM / Bitcoin / Sui RPC providers | Balances, send, history fallback | Public addresses, transaction queries |
| **Helius** (optional â€” only if you paste an API key) | Enhanced Solana History when you open History / Refresh | Public address, history queries |
| Jupiter Price / token metadata APIs | Holdings USD prices | Token mint IDs for assets you hold |
| CoinGecko (and rare fallbacks) | Native coin prices / charts | Public coin identifiers |
| WalletConnect / Reown relays | WalletConnect pairing when you use WC | Session / URI metadata you initiate |
| Block explorers | Open transaction / address links | Public addresses / transaction ids |
| Websites you visit | dApp use after you connect | Controlled by those sites' own policies |

Third parties may process technical data such as IP address and timestamps as with ordinary web traffic. Smart Wallet does not control third-party retention policies.

## Permissions (plain language)

| Permission | Why |
|------------|-----|
| Storage | Save local wallet state and settings |
| Clipboard | Copy addresses; optional paste-safety checks on Send / Bridge |
| Access to websites / scripting | Host permission may be broad for RPC flexibility; **content-script inject** of the wallet provider runs only on an **allowlisted** set of DEX / DeFi hosts (see DOCUMENTATION Â§14.2). Connect and sign still require your approval. |
| Tabs | Open or focus the wallet UI for approvals and Ledger |
| Alarms | Auto-lock after inactivity when password protection is on |
| Offscreen | Local helper used for signing-related work |
| HID | Communicate with Ledger hardware wallets over USB |

## Security notes (non-exhaustive)

- Software keys are encrypted at rest with AES-GCM (user password when protection is on; device wrap key when protection is off). Plain seeds load only briefly for sign / approve / intentional reveal. Removing a wallet seed requires explicit âœ• Remove on that account.  
- Ledger private keys remain on the hardware device.  
- Always verify dApp origins, amounts, and addresses before approving.  
- Cryptocurrency involves risk of permanent loss of funds.

## Children

Smart Wallet is not directed at children under 13 (or the minimum age required in your jurisdiction).

## Changes

We may update this policy. The "Last updated" date will change when we do. Continued use after an update means you accept the revised policy.

## Contact

| Role | Address |
|------|---------|
| **Developer** | [Whitebrick86@gmail.com](mailto:Whitebrick86@gmail.com) |
| **Support** | [smartwallethelp@outlook.com](mailto:smartwallethelp@outlook.com) |

Full contact page: [CONTACTS.md](./CONTACTS.md)

You may also open an issue on the documentation repository:

**https://github.com/whitebrick86-jpg/Smart-Wallet**

Do not email seed phrases or private keys.
