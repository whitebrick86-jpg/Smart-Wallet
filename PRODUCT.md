# Smart Wallet — product overview

Smart Wallet is a non-custodial multi-chain browser wallet for Chrome and Opera (Manifest V3).

Users create or import accounts, view balances, send, swap, and bridge, and connect to web dApps with standard wallet APIs. Ledger hardware wallets are supported for Solana and EVM. Keys stay on the device; Smart Wallet does not take custody of seeds or funds.

## Networks

Solana, Ethereum, Base, Polygon, BNB Smart Chain, Robinhood ETH, Arbitrum One, Optimism, Avalanche C-Chain, Bitcoin, and Sui.

## Fees

Smart Wallet charges a platform fee only on its internal swap and bridge paths. Send and external dApp activity have no Smart Wallet platform fee.

| Path | Smart Wallet platform fee |
|------|---------------------------|
| Internal swap | 0.45% |
| Internal bridge | 0.85% |
| Send / external DEX or bridge | 0% |

LiFi-routed EVM quotes may also show LiFi’s own service fee. That amount is not Smart Wallet revenue. Full copy: [FEE-DISCLOSURE.md](./Chrome-extension-store-for-reviewers/FEE-DISCLOSURE.md).

## Privacy and permissions

- No Smart Wallet cloud custody of seeds or keys
- Page-provider inject is limited to an allowlist of dApp hosts
- `https://*/*` and `wss://*/*` are **optional** and used only for a user-selected custom RPC origin

See [PRIVACY-POLICY.md](./Chrome-extension-store-for-reviewers/PRIVACY-POLICY.md) and [HOST-PERMISSIONS.md](./Chrome-extension-store-for-reviewers/HOST-PERMISSIONS.md).

## Support

[CONTACTS.md](./Chrome-extension-store-for-reviewers/CONTACTS.md)
