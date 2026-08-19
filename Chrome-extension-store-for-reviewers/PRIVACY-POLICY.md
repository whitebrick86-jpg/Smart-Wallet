# Smart Wallet — Privacy Policy

**Last updated:** 2026-08-19

## Who we are

Smart Wallet is a non-custodial browser extension that lets users hold and use cryptocurrency keys on their device, interact with supported blockchain networks, and connect to decentralized applications (“dApps”).

**Documentation:** https://github.com/whitebrick86-jpg/Smart-Wallet

## Non-custodial design

Smart Wallet does not take custody of your cryptocurrency.

* We do **not** operate a Smart Wallet cloud account that stores your seed phrase or private keys.
* We do **not** intentionally transmit seed phrases, private keys, wallet passwords, or decrypted vault contents to Smart Wallet servers.
* Smart Wallet infrastructure cannot approve or sign transactions for you.
* Software-wallet transaction signing occurs locally on your device.
* Ledger private keys remain on the Ledger hardware device.
* We do **not** sell personal information for advertising.
* We do **not** use wallet activity for personalized advertising.

Never send your seed phrase, private keys, password, or recovery information to Smart Wallet support.

## Information stored on your device

Smart Wallet stores wallet-related information locally in your browser profile. This may include:

* Public wallet addresses and account labels
* Encrypted vault data for software wallets
* Wallet and interface settings
* Selected blockchain network
* Auto-lock preferences
* Optional Custom RPC configuration
* Optional RPC credentials that you choose to enter
* Recent transaction-history rows
* Balance, token, metadata, and price caches
* WalletConnect session information
* dApp permissions and connection records
* Pending-transaction and nonce information
* A non-exportable Managed RPC device-authentication key
* Managed RPC registration status and related security metadata

Software-wallet vault data is encrypted at rest. Private wallet keys and seed phrases are not included in Managed RPC registration.

Removing the extension or clearing its extension storage removes locally stored information from that browser profile. Clearing local storage may cause Smart Wallet to create a new Managed RPC device identity if the extension is used again.

Removing local extension data does not necessarily delete security, rate-limit, or revoked-device records previously stored by Smart Wallet infrastructure.

## Silent Managed RPC device registration

Smart Wallet may silently create a unique, non-exportable ECDSA P-256 authentication key for the installation.

This device key is separate from your cryptocurrency wallet keys.

* The private device-authentication key remains on the device and is not intentionally exported.
* Only the corresponding public authentication key and a derived device identifier are sent to Smart Wallet infrastructure.
* The server assigns ordinary Managed RPC permission identified as `rpc:use`.
* Ordinary installations do not receive administrative permission.
* Device registration does not provide Smart Wallet with access to your seed phrase, private keys, password, encrypted vault, or funds.
* Device registration is used to authenticate Managed RPC requests, apply security controls, prevent replay, enforce rate limits, investigate abuse, and maintain service reliability.

Automatic registration does not prove that a caller is a genuine Chrome installation. Smart Wallet therefore also uses server-side rate limits, request-signature checks, replay protection, device status, concurrency controls, and other abuse-prevention measures.

A revoked device identity cannot use Managed RPC. Clearing browser storage may generate a different device identity, but the previous server record may remain for security and abuse-prevention purposes.

## RPC routing and the production manager

Smart Wallet may route supported blockchain requests through:

1. Built-in public RPC providers;
2. A Custom RPC provider configured by the user; or
3. Developer-managed production RPC infrastructure.

An authorized Smart Wallet operator may use a secured production manager to select the global RPC-routing mode:

* **Managed RPC:** eligible Smart Wallet installations use developer-managed RPC infrastructure for supported requests.
* **Public RPC:** Smart Wallet uses built-in public RPC providers, subject to any applicable user-configured Custom RPC setting.

The active routing mode may be changed to support reliability, security, maintenance, provider availability, capacity management, or cost control.

The production manager:

* Changes the RPC route used for supported blockchain requests.
* Does not download or execute remote extension code.
* Does not provide access to seed phrases, private keys, passwords, or decrypted vault contents.
* Does not allow the operator to approve, create, initiate, or sign transactions for a user.
* Does not give ordinary wallet installations permission to change the global RPC mode.
* Does not convert a customer device identity into an administrator.
* Does not override the requirement for user approval and local signing.
* Does not remotely disable a user-configured Custom RPC solely because Managed RPC is disabled.

Administrative authorization is kept separate from ordinary Managed RPC authorization. Only a separately authorized owner device or secured administrative credential may change the global RPC-routing mode.

The production-manager URL, administrative credentials, provider credentials, internal quota thresholds, and security-sensitive abuse controls are not included in customer extension packages.

## Information processed during RPC requests

When Managed RPC is active, Smart Wallet infrastructure and its infrastructure providers may process information necessary to complete, secure, troubleshoot, rate-limit, and maintain RPC service, including:

* Selected blockchain network
* RPC method
* RPC parameters
* Public wallet addresses
* Public token or contract addresses
* Transaction hashes
* Signed transaction data submitted for broadcasting
* Block, balance, nonce, fee, gas, receipt, history, and token queries
* Device identifier or device-key fingerprint
* Request-signature and replay-protection metadata
* Correlation or request identifiers
* IP address
* Timestamps
* Response status
* Latency, retry, quota, and error information
* Provider-routing and failover results

A signed transaction contains information intended for submission to a blockchain, including its public sender, destination, amount or contract call, signature, and other transaction fields. Submission of a signed transaction does not disclose the private key used to create its signature.

Smart Wallet does not require or intentionally transmit a seed phrase or private key as part of an RPC request.

When Public RPC or Custom RPC is active, applicable requests may be sent directly from the browser to the selected third-party provider. That provider may process the request according to its own privacy and retention practices.

## Active-network request controls

During normal wallet use, Smart Wallet limits automatic blockchain requests to the currently selected network.

Additional networks may be contacted temporarily when necessary for an explicit user-facing feature, including:

* Viewing balances across supported networks
* Performing a cross-chain bridge
* Using a swap on a selected network
* Responding to an authorized dApp request
* Monitoring confirmation of an already-submitted transaction
* Validating a Custom RPC endpoint
* Refreshing a network explicitly selected by the user

Smart Wallet uses caching, request deduplication, bounded concurrency, sequential provider failover, retry limits, and inactive-network controls to reduce unnecessary network traffic.

Opening an explicit all-network balance view may cause Smart Wallet to refresh balances for the networks displayed in that view. Closing the view ends that temporary refresh activity, subject to bounded transaction-confirmation monitoring or another active user-requested operation.

## Other network requests

When you use Smart Wallet, the extension may contact third-party services required for its wallet features.

| Destination or service                                                       | Purpose                                                                     | Typical information                                                                                                                 |
| ---------------------------------------------------------------------------- | --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Smart Wallet Managed RPC infrastructure                                      | Authenticated blockchain access, reliability, quotas, and abuse prevention  | Device public key or identifier, IP address, chain, RPC method and parameters, public addresses, transaction data, request metadata |
| Public Solana, EVM, Bitcoin, and Sui RPC providers                           | Balances, history, fees, transaction submission, confirmation, and fallback | Public addresses, transaction data, RPC methods and parameters, IP address                                                          |
| User-configured Custom RPC provider                                          | Blockchain access selected by the user                                      | Public addresses, transaction data, RPC methods and parameters, IP address                                                          |
| Helius, if configured or used for supported Solana features                  | Solana RPC or enhanced history                                              | Public Solana addresses, transaction and history queries, IP address                                                                |
| Jupiter and supported token or price services                                | Solana swaps, token information, holdings prices, and metadata              | Token mint addresses, public transaction information, quote parameters, IP address                                                  |
| CoinGecko and supported price-data fallbacks                                 | Native-asset prices and charts                                              | Public asset identifiers and IP address                                                                                             |
| LiFi-related Smart Wallet infrastructure and supported bridge/swap providers | Quotes, routes, transaction construction, status, swaps, and bridges        | Source/destination chains, tokens, amounts, public addresses, route and transaction information, IP address                         |
| WalletConnect/Reown relays                                                   | WalletConnect pairing and sessions initiated by the user                    | Pairing, session, chain, account, and request metadata                                                                              |
| On-ramp providers opened by the user                                         | Purchasing cryptocurrency                                                   | Information entered or provided directly to the on-ramp provider                                                                    |
| Block explorers                                                              | Address and transaction links opened by the user                            | Public addresses, transaction hashes, IP address                                                                                    |
| Connected dApps and websites                                                 | dApp connections and requests authorized by the user                        | Public account, selected network, permissions, and approved request information                                                     |

Third-party providers may receive ordinary web-request information such as IP address, timestamp, browser/network metadata, and the contents of requests sent to them. Their collection, use, security, and retention practices are governed by their own policies.

## How information is used

Smart Wallet uses information only as reasonably necessary to:

* Provide wallet features requested by the user
* Retrieve balances, prices, tokens, and transaction history
* Construct, submit, and monitor user-approved transactions
* Provide swaps, bridges, WalletConnect, Ledger, dApp, and on-ramp features
* Route requests between Managed, Public, and Custom RPC providers
* Authenticate eligible Managed RPC devices
* Apply per-device, per-IP, per-chain, per-method, concurrency, broadcast, and global limits
* Prevent replay, fraud, abuse, duplicate broadcasts, and unauthorized administrative actions
* Diagnose failures and maintain performance, security, and reliability
* Comply with applicable legal obligations

Smart Wallet does not use this information to independently approve or sign cryptocurrency transactions.

## Information sharing

Information may be shared with infrastructure providers only when necessary to provide, secure, maintain, or troubleshoot Smart Wallet’s disclosed wallet functionality.

This may include:

* Blockchain RPC providers
* Cloud infrastructure providers
* Swap and bridge providers
* Price and token-information providers
* WalletConnect/Reown
* Block explorers
* On-ramp providers selected by the user
* Security or abuse-prevention service providers

Smart Wallet may also disclose information when reasonably necessary to comply with applicable law, respond to valid legal process, or investigate fraud, abuse, or security incidents.

Smart Wallet does not sell personal information or transfer wallet activity for personalized advertising.

## Data retention

Local extension data remains in the browser profile until it is deleted, replaced, cleared, or removed according to wallet functionality and browser behavior.

Smart Wallet infrastructure may retain device-registration, revocation, security, quota, request, and operational records for as long as reasonably necessary to:

* Provide Managed RPC service
* Maintain device authorization
* Prevent replay and abuse
* Enforce quotas and capacity limits
* Investigate security or reliability incidents
* Meet legal obligations

Third-party providers determine their own retention periods under their respective privacy policies.

## Permissions in plain language

| Permission                     | Purpose                                                                                                        |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| Storage                        | Save encrypted wallet state, settings, permissions, caches, and local registration state                       |
| Clipboard                      | Copy addresses and perform supported paste-safety checks                                                       |
| Access to websites / scripting | Inject the Smart Wallet provider on supported allowlisted dApp hosts so users can connect and approve requests |
| Tabs                           | Open or focus the wallet for approvals, dApp requests, and Ledger workflows                                    |
| Alarms                         | Support auto-lock, bounded maintenance, and scheduled wallet tasks                                             |
| Offscreen                      | Provide a local helper for supported signing-related work                                                      |
| HID                            | Communicate with Ledger hardware wallets over USB                                                              |

Smart Wallet should request only permissions reasonably necessary for its disclosed wallet functionality.

## Security

Smart Wallet uses security measures intended to protect wallet and service operations, including:

* Local encrypted vault storage
* Non-exportable Managed RPC device-authentication keys
* Signed Managed RPC requests
* Timestamp and nonce validation
* Body-hash and request binding
* Replay protection
* Device revocation
* Rate limits and bounded concurrency
* Method and chain allowlists
* Transaction-broadcast protections
* Separation between customer RPC permission and administrative permission
* HTTPS or WSS for supported external transmissions

No system is completely secure. Users remain responsible for protecting their device, password, seed phrase, private keys, Ledger device, recovery information, and transaction approvals.

Always verify dApp origins, networks, tokens, amounts, fees, and destination addresses before approving a transaction.

Cryptocurrency transactions may be irreversible and may result in permanent loss.

## User choices

Users may:

* Select supported blockchain networks
* Configure a supported Custom RPC endpoint
* Choose whether to use optional third-party features
* Disconnect dApps and WalletConnect sessions
* Remove local accounts or wallet data through supported wallet controls
* Clear extension storage or remove the extension
* Decline to approve or sign a transaction

Some wallet features require network requests. Disabling required connectivity may prevent those features from functioning.

## Chrome Web Store Limited Use

Smart Wallet limits the use of information obtained through extension permissions and wallet functionality to providing, maintaining, securing, and improving the extension’s disclosed single purpose as a non-custodial cryptocurrency wallet.

Smart Wallet does not use or transfer user information for personalized advertising, creditworthiness, lending decisions, or sale to data brokers.

Smart Wallet’s use of information is intended to comply with the Chrome Web Store User Data Policy, including its Limited Use requirements.

This privacy policy does not replace any prominent in-product disclosure or affirmative consent required before Smart Wallet begins a materially different data practice.

## Children

Smart Wallet is not directed to children under 13 or the minimum age required in the user’s jurisdiction.

## Changes to this policy

We may update this policy when Smart Wallet’s functionality, infrastructure, providers, legal obligations, or data practices change.

The “Last updated” date will be revised when the policy changes. Material changes to data practices should also be disclosed through the extension or another appropriate user-facing notice when required.

## Contact

| Role          | Address                                                           |
| ------------- | ----------------------------------------------------------------- |
| **Developer** | [Whitebrick86@gmail.com](mailto:Whitebrick86@gmail.com)           |
| **Support**   | [smartwallethelp@outlook.com](mailto:smartwallethelp@outlook.com) |

Full contact page: [CONTACTS.md](./CONTACTS.md)

You may also open an issue on the documentation repository:

https://github.com/whitebrick86-jpg/Smart-Wallet

Do not email seed phrases, private keys, passwords, authentication tokens, or recovery information.
