# Smart Wallet — Privacy Policy

**Last updated:** 2026-08-20

## Who we are

Smart Wallet is a non-custodial browser extension that lets users hold and use cryptocurrency keys on their device, interact with supported blockchain networks, connect to decentralized applications (“dApps”), and use optional wallet-address messaging features.

**Documentation:** https://github.com/whitebrick86-jpg/Smart-Wallet

## Non-custodial design

Smart Wallet does not take custody of your cryptocurrency.

* We do **not** operate a Smart Wallet cloud account that stores your seed phrase or private keys.
* We do **not** intentionally transmit seed phrases, private keys, wallet passwords, or decrypted vault contents to Smart Wallet servers.
* Smart Wallet infrastructure cannot approve or sign transactions for you.
* Software-wallet transaction signing occurs locally on your device.
* Ledger private keys remain on the Ledger hardware device.
* Messaging authentication does not disclose your seed phrase or private wallet key.
* We do **not** sell personal information for advertising.
* We do **not** use wallet activity or message content for personalized advertising.

Never send your seed phrase, private keys, wallet password, recovery information, authentication codes, or administrative credentials through Smart Wallet messaging or to Smart Wallet support.

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
* A non-exportable infrastructure device-authentication key
* Managed RPC registration status and related security metadata
* Messaging conversations and locally displayed message copies
* Outgoing-message delivery states such as Sending, Sent, or Not delivered
* Cached announcements
* Messaging refresh and synchronization metadata
* Local privacy-consent records

Software-wallet vault data is encrypted at rest. Private wallet keys and seed phrases are not included in Managed RPC registration or messaging registration.

Removing the extension or clearing its extension storage removes locally stored information from that browser profile. Clearing local storage may cause Smart Wallet to create a new infrastructure device identity if the extension is used again.

Removing local extension data does not necessarily delete messages, announcements, security records, rate-limit records, revoked-device records, or other information previously stored by Smart Wallet infrastructure.

## Wallet-to-wallet messaging

Smart Wallet may provide an optional messaging system that lets one wallet address send a message to another wallet address.

This messaging system does not import, merge, transfer, or provide access to the recipient’s wallet. The recipient proves control of the receiving address through a signed authentication request before Smart Wallet infrastructure returns messages addressed to that wallet.

### Information processed for messaging

When a message is sent, stored, retrieved, or replied to, Smart Wallet infrastructure may process and store:

* Sender public wallet address
* Recipient public wallet address
* Message body
* Message or thread identifier
* Reply or conversation metadata
* Message creation and storage timestamps
* Message retrieval or synchronization metadata
* Request identifiers
* Cryptographic authentication information
* Public authentication keys or derived identifiers
* IP address
* Rate-limit, replay-prevention, security, error, and diagnostic metadata

Wallet addresses and blockchain transaction information are generally public. However, a message body, the relationship between addresses, an IP address, and associated metadata may constitute personal information depending on its contents and applicable law.

Do not include seed phrases, private keys, wallet passwords, recovery information, authentication tokens, sensitive financial information, or information you do not want stored by Smart Wallet infrastructure.

### Authentication and inbox access

Inbox retrieval requires a signed request intended to demonstrate control of the receiving wallet address.

The authentication signature:

* Is created locally using the applicable wallet or device authentication mechanism.
* Does not reveal the private key used to create it.
* Is bound to request information such as the address, timestamp, nonce, or request body.
* May be checked for freshness and replay.
* Does not authorize Smart Wallet to sign cryptocurrency transactions.
* Does not give another user access to the receiving wallet.
* Does not grant ordinary users administrative permission.

Unsigned inbox lookup is not supported. A user should not be able to retrieve another address’s inbox merely by knowing that public address.

No authentication system is perfect. Users remain responsible for securing their device, wallet, passwords, private keys, recovery information, and active sessions.

### Message encryption and operator access

Smart Wallet messaging is protected using encrypted network connections, such as HTTPS, while messages are transmitted.

Smart Wallet’s cloud infrastructure provider encrypts stored messaging values at rest. Encryption and decryption of infrastructure values are handled by the infrastructure provider and authorized Worker processes.

**Smart Wallet messages are not end-to-end encrypted.**

This means:

* Message bodies are stored by Smart Wallet infrastructure.
* Authorized Smart Wallet infrastructure can technically process or access message bodies when necessary to store, retrieve, secure, troubleshoot, moderate, or operate the service.
* Infrastructure encryption at rest does not prevent authorized server-side processing.
* Users should not treat Smart Wallet messaging as a private, end-to-end encrypted communications service.
* Smart Wallet does not claim that only the sender and recipient can technically access message contents.

Access to stored message content should be limited to circumstances reasonably necessary for service operation, security, abuse investigation, legal compliance, or technical support. Smart Wallet does not use message bodies for personalized advertising or sell them to data brokers.

### Message-delivery status

Smart Wallet may display the following outgoing-message states:

* **Sending:** The request is still in progress.
* **Sent:** Smart Wallet infrastructure confirmed storage of the message and its recipient inbox index.
* **Not delivered:** The request failed because of a network, authorization, quota, storage, validation, or server error.

“Sent” does not mean that the recipient opened, read, or retrieved the message.

Smart Wallet currently does not claim “Delivered” or “Read” unless an authenticated recipient acknowledgment system is implemented and successfully confirms that event.

A failed outgoing message may remain stored locally on the sender’s device with a Not delivered status even though it was not stored by the server.

Smart Wallet does not automatically retry failed sends where doing so could create duplicate messages.

## Announcements

Smart Wallet may retrieve operator announcements intended for all supported wallet installations.

Announcements may contain:

* Service information
* Security notices
* Maintenance notices
* Feature information
* Warnings
* Other information related to Smart Wallet operation

Announcement titles, bodies, identifiers, and timestamps may be stored by Smart Wallet infrastructure and cached locally by the extension.

Customer builds may read announcements but do not receive administrative authority to create, edit, or delete global announcements.

Creating, editing, or deleting an announcement requires separate administrator or authorized owner-device authentication. Ordinary users and ordinary `rpc:use` devices are not authorized to publish global announcements.

Announcements:

* Do not provide the operator with access to user wallets.
* Cannot approve or sign transactions.
* Must not request seed phrases, private keys, passwords, or recovery information.
* Are not blockchain transactions.
* May be cached locally to reduce repeated network and storage requests.

Users should treat any announcement requesting a seed phrase, private key, wallet password, authentication code, or transfer of funds as suspicious.

## Messaging and announcement network behavior

Smart Wallet limits messaging network activity to reduce unnecessary requests.

Subject to the active version and feature configuration:

* Inbox messages are retrieved when the user opens Inbox or manually requests an inbox refresh.
* Home, Send, Swap, and Bridge should not independently retrieve the Inbox.
* Announcements may be retrieved when a visible, unlocked wallet opens or unlocks.
* Announcements may be cached locally for a limited period.
* Announcement refreshes should stop while the wallet is hidden or locked.
* A manual refresh may bypass a previous cached result.
* Multiple extension windows in the same browser profile may share an announcement cache.
* The operator’s publication of an announcement may update the owner wallet’s local cache.

Network behavior may change as Smart Wallet improves reliability, security, or performance. Material changes to data collection will be disclosed when required.

## Silent infrastructure device registration

Smart Wallet may silently create a unique, non-exportable ECDSA P-256 authentication key for the installation.

This device key is separate from your cryptocurrency wallet keys.

* The private device-authentication key remains on the device and is not intentionally exported.
* Only the corresponding public authentication key and a derived device identifier are sent to Smart Wallet infrastructure.
* The server assigns ordinary infrastructure permission identified as `rpc:use`.
* Ordinary installations do not receive administrative permission.
* Device registration does not provide Smart Wallet with access to your seed phrase, private keys, password, encrypted vault, or funds.
* Device registration may be used to authenticate Managed RPC or supported infrastructure requests, apply security controls, prevent replay, enforce rate limits, investigate abuse, and maintain service reliability.

Automatic registration does not prove that a caller is a genuine Chrome installation. Smart Wallet therefore also uses server-side rate limits, request-signature checks, replay protection, device status, concurrency controls, and other abuse-prevention measures.

A revoked device identity cannot use protected infrastructure features. Clearing browser storage may generate a different device identity, but the previous server record may remain for security and abuse-prevention purposes.

## RPC routing and the production manager

Smart Wallet may route supported blockchain requests through:

1. Built-in public RPC providers;
2. A Custom RPC provider configured by the user; or
3. Developer-managed production RPC infrastructure.

An authorized Smart Wallet operator may use a secured production manager to select the global RPC-routing mode:

* **Managed RPC:** Eligible Smart Wallet installations use developer-managed RPC infrastructure for supported requests.
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

Administrative authorization is kept separate from ordinary Managed RPC authorization. Only a separately authorized owner device or secured administrative credential may change the global RPC-routing mode or perform other protected administrative operations.

The production-manager URL, administrative credentials, provider credentials, internal quota thresholds, and security-sensitive abuse controls are not included in customer extension packages.

Messaging and announcements may be enabled or disabled independently of Managed RPC. Activating messaging does not activate Managed RPC, and switching RPC mode does not grant customers announcement-administration rights.

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

When you use Smart Wallet, the extension may contact Smart Wallet infrastructure and third-party services required for its features.

| Destination or service                                                       | Purpose                                                                     | Typical information                                                                                                                     |
| ---------------------------------------------------------------------------- | --------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Smart Wallet messaging infrastructure                                        | Store and retrieve wallet-address messages                                  | Sender and recipient public addresses, message body, identifiers, timestamps, authentication proof, IP address, and security metadata   |
| Smart Wallet announcement infrastructure                                     | Retrieve and administer service announcements                               | Announcement title, body, identifier, timestamp, IP address, cache and request metadata                                                 |
| Smart Wallet Managed RPC infrastructure                                      | Authenticated blockchain access, reliability, quotas, and abuse prevention  | Device public key or identifier, IP address, chain, RPC method and parameters, public addresses, transaction data, and request metadata |
| Public Solana, EVM, Bitcoin, and Sui RPC providers                           | Balances, history, fees, transaction submission, confirmation, and fallback | Public addresses, transaction data, RPC methods and parameters, and IP address                                                          |
| User-configured Custom RPC provider                                          | Blockchain access selected by the user                                      | Public addresses, transaction data, RPC methods and parameters, and IP address                                                          |
| Helius, if configured or used for supported Solana features                  | Solana RPC or enhanced history                                              | Public Solana addresses, transaction and history queries, and IP address                                                                |
| Jupiter and supported token or price services                                | Solana swaps, token information, holdings prices, and metadata              | Token mint addresses, public transaction information, quote parameters, and IP address                                                  |
| CoinGecko and supported price-data fallbacks                                 | Native-asset prices and charts                                              | Public asset identifiers and IP address                                                                                                 |
| LiFi-related Smart Wallet infrastructure and supported bridge/swap providers | Quotes, routes, transaction construction, status, swaps, and bridges        | Source and destination chains, tokens, amounts, public addresses, route and transaction information, and IP address                     |
| WalletConnect/Reown relays                                                   | WalletConnect pairing and sessions initiated by the user                    | Pairing, session, chain, account, and request metadata                                                                                  |
| On-ramp providers opened by the user                                         | Purchasing cryptocurrency                                                   | Information entered or provided directly to the on-ramp provider                                                                        |
| Block explorers                                                              | Address and transaction links opened by the user                            | Public addresses, transaction hashes, and IP address                                                                                    |
| Connected dApps and websites                                                 | dApp connections and requests authorized by the user                        | Public account, selected network, permissions, and approved request information                                                         |

Third-party providers may receive ordinary web-request information such as IP address, timestamp, browser or network metadata, and the contents of requests sent to them. Their collection, use, security, and retention practices are governed by their own policies.

## How information is used

Smart Wallet uses information only as reasonably necessary to:

* Provide wallet features requested by the user
* Store, route, retrieve, and display wallet-address messages
* Authenticate inbox access
* Display service and security announcements
* Maintain accurate outgoing-message status
* Prevent unauthorized inbox access and announcement publication
* Retrieve balances, prices, tokens, and transaction history
* Construct, submit, and monitor user-approved transactions
* Provide swaps, bridges, WalletConnect, Ledger, dApp, and on-ramp features
* Route requests between Managed, Public, and Custom RPC providers
* Authenticate eligible infrastructure devices
* Apply per-device, per-IP, per-chain, per-method, concurrency, broadcast, messaging, and global limits
* Prevent replay, fraud, spam, abuse, duplicate submissions, duplicate broadcasts, and unauthorized administrative actions
* Diagnose failures and maintain performance, security, and reliability
* Comply with applicable legal obligations

Smart Wallet does not use messaging, RPC, or operational information to independently approve or sign cryptocurrency transactions.

## Information sharing

Information may be shared with infrastructure providers only when necessary to provide, secure, maintain, or troubleshoot Smart Wallet’s disclosed functionality.

This may include:

* Cloud infrastructure and storage providers
* Blockchain RPC providers
* Swap and bridge providers
* Price and token-information providers
* WalletConnect/Reown
* Block explorers
* On-ramp providers selected by the user
* Security or abuse-prevention service providers

Message bodies may be processed by the cloud infrastructure used to store and retrieve them. Smart Wallet does not sell message bodies, wallet activity, or personal information for personalized advertising.

Smart Wallet may disclose information when reasonably necessary to:

* Comply with applicable law or valid legal process
* Protect users, Smart Wallet, or the public
* Investigate fraud, spam, abuse, security incidents, or service misuse
* Enforce applicable service rules
* Establish, exercise, or defend legal claims

Public blockchain information remains subject to the permanent and public nature of the applicable blockchain.

## Data retention

Local extension data remains in the browser profile until it is deleted, replaced, cleared, or removed according to wallet functionality and browser behavior.

Server-stored wallet messages may remain on Smart Wallet infrastructure until they are deleted under the service’s operational deletion process, removed in response to an applicable verified request, removed for security or abuse reasons, or deleted when the messaging service or applicable storage record is retired.

Unless the product provides a confirmed deletion function, users should not assume that deleting a local conversation, clearing browser storage, removing an account from the extension, or uninstalling Smart Wallet immediately deletes the server-stored copy.

Smart Wallet should retain message bodies and associated metadata only for as long as reasonably necessary to:

* Deliver and synchronize messages
* Maintain inbox integrity
* Prevent duplicates, replay, spam, fraud, and abuse
* Investigate security or reliability incidents
* Comply with legal obligations
* Resolve verified deletion or support requests

Announcements may be retained while they remain relevant to users, for operational records, or as required for security and legal purposes. Locally cached announcements expire or are replaced according to extension caching behavior.

Smart Wallet infrastructure may retain device-registration, revocation, security, quota, request, authentication, and operational records for as long as reasonably necessary to:

* Provide Managed RPC and messaging services
* Maintain device and inbox authorization
* Prevent replay, spam, fraud, and abuse
* Enforce quotas and capacity limits
* Investigate security or reliability incidents
* Meet legal obligations

Backups, security records, and provider-managed replicas may persist for a limited period after deletion from active systems.

Third-party providers determine their own retention periods under their respective privacy policies.

## User requests and deletion

Depending on applicable law, users may request information about, correction of, or deletion of personal information controlled by Smart Wallet.

Because Smart Wallet does not operate conventional named user accounts, Smart Wallet may need reasonable cryptographic proof that the requester controls the relevant wallet address before acting on a message-related request.

Smart Wallet will not ask for a seed phrase, private key, or wallet password to verify a request.

Certain information may be retained where reasonably necessary for security, fraud prevention, legal compliance, dispute resolution, or protection of other users.

Deleting server-stored messages for one address may affect the other participant’s access or locally stored copy. Information independently retained on another user’s device cannot necessarily be deleted by Smart Wallet.

Requests may be sent using the contact information below.

## Permissions in plain language

| Permission                     | Purpose                                                                                                                |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| Storage                        | Save encrypted wallet state, settings, permissions, messaging state, announcement caches, and local registration state |
| Clipboard                      | Copy addresses and perform supported paste-safety checks                                                               |
| Access to websites / scripting | Inject the Smart Wallet provider on supported allowlisted dApp hosts so users can connect and approve requests         |
| Tabs                           | Open or focus the wallet for approvals, dApp requests, and Ledger workflows                                            |
| Alarms                         | Support auto-lock, bounded maintenance, and scheduled wallet tasks                                                     |
| Offscreen                      | Provide a local helper for supported signing-related work                                                              |
| HID                            | Communicate with Ledger hardware wallets over USB                                                                      |

Smart Wallet should request only permissions reasonably necessary for its disclosed wallet functionality.

## Security

Smart Wallet uses security measures intended to protect wallet and service operations, including:

* Local encrypted vault storage
* Infrastructure-provider encryption of stored cloud values
* Encrypted HTTPS or WSS transmissions
* Non-exportable infrastructure device-authentication keys
* Signed inbox and Managed RPC requests
* Timestamp and nonce validation
* Body-hash and request binding
* Replay protection
* Device revocation
* Rate limits and bounded concurrency
* Method and chain allowlists
* Transaction-broadcast protections
* Separation between customer permission and administrative permission
* Administrator-only announcement mutations
* Fail-closed handling for unavailable authentication or storage

These safeguards do not make Smart Wallet messaging end-to-end encrypted.

No system is completely secure. Users remain responsible for protecting their device, password, seed phrase, private keys, Ledger device, recovery information, active sessions, and transaction approvals.

Always verify dApp origins, networks, tokens, amounts, fees, and destination addresses before approving a transaction.

Cryptocurrency transactions may be irreversible and may result in permanent loss.

## User choices

Users may:

* Choose whether to use wallet-to-wallet messaging
* Avoid sending sensitive information through messaging
* Refresh an Inbox or announcement list
* Select supported blockchain networks
* Configure a supported Custom RPC endpoint
* Choose whether to use optional third-party features
* Disconnect dApps and WalletConnect sessions
* Remove local accounts or wallet data through supported wallet controls
* Clear extension storage or remove the extension
* Decline to approve or sign a transaction
* Contact Smart Wallet regarding applicable privacy or deletion requests

A recipient may receive a message addressed to a public wallet address without first accepting an invitation. Opening or retrieving the Inbox causes the wallet to request messages addressed to that wallet.

Some wallet features require network requests. Disabling required connectivity may prevent those features from functioning.

## Chrome Web Store Limited Use

Smart Wallet limits the use of information obtained through extension permissions and wallet functionality to providing, maintaining, securing, and improving the extension’s disclosed single purpose as a non-custodial cryptocurrency wallet, including its wallet-address communication and service-notice features.

Smart Wallet does not use or transfer user information for:

* Personalized advertising
* Creditworthiness or lending decisions
* Sale to data brokers
* Unrelated commercial profiling

Smart Wallet’s use of information is intended to comply with the Chrome Web Store User Data Policy, including its Limited Use requirements.

This privacy policy does not replace any prominent in-product disclosure or affirmative consent required before Smart Wallet begins a materially different data practice.

## International processing

Smart Wallet infrastructure and service providers may process information in countries other than the user’s country. Those countries may have data-protection laws that differ from the laws where the user resides.

Smart Wallet will use applicable safeguards where required by law.

Users should not rely on Smart Wallet messaging for emergency communications or communications that must remain available in every country or region. Service availability may be affected by network restrictions, infrastructure availability, local law, or provider reachability.

## Children

Smart Wallet is not directed to children under 13 or the minimum age required in the user’s jurisdiction.

Do not knowingly use Smart Wallet messaging to collect personal information from children contrary to applicable law.

## Changes to this policy

We may update this policy when Smart Wallet’s functionality, infrastructure, providers, legal obligations, or data practices change.

The “Last updated” date will be revised when the policy changes. Material changes to data practices should also be disclosed through the extension or another appropriate user-facing notice when required.

Continued use after an update may be subject to any notice or consent required by applicable law or platform policy.

## Contact

| Role          | Address                                                           |
| ------------- | ----------------------------------------------------------------- |
| **Developer** | [Whitebrick86@gmail.com](mailto:Whitebrick86@gmail.com)           |
| **Support**   | [smartwallethelp@outlook.com](mailto:smartwallethelp@outlook.com) |

Full contact page: [CONTACTS.md](./CONTACTS.md)

You may also open an issue on the documentation repository:

https://github.com/whitebrick86-jpg/Smart-Wallet

Do not email or message seed phrases, private keys, wallet passwords, authentication tokens, administrative credentials, or recovery information.
