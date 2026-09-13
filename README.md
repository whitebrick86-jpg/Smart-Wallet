# Smart Wallet

**Documentation-only repository.**

User and architecture documentation for **Smart Wallet** (Chrome / Opera MV3 extension). Extension source code is **not** published in this repo.

**Docs / privacy:** this repository · **Live product:** **0.11.698** · **Product map:** [PRODUCT.md](./PRODUCT.md)

## Chrome Web Store reviewers

**→ [Chrome-extension-store-for-reviewers/](./Chrome-extension-store-for-reviewers/)** — privacy, privacy policy, fee disclosure, host-permissions justification, and contacts (**canonical** — no duplicate copies at repo root).

## Documents

| File | Description |
|------|-------------|
| **[PRODUCT.md](./PRODUCT.md)** | **Canonical product map:** extension + LiFi Worker folders, staging URL, fee classes, no API key in the extension |
| **[MESSAGING.md](./MESSAGING.md)** | Inbox / Messaging: folders, buttons, Delete conversation vs Delete for me vs server deletion request |
| **[Chrome-extension-store-for-reviewers/](./Chrome-extension-store-for-reviewers/)** | **CWS reviewer pack** (privacy, fees, host permissions, contacts) |
| **[ARCHITECTURE.md](./ARCHITECTURE.md)** | Build architecture: surfaces, storage, signing, live feeds, dApp inject, fees, security, packaging |
| **[CHAINS.md](./CHAINS.md)** | **Networks:** current 12-chain list, including Sonic |
| **[NONCE-GUARD.md](./NONCE-GUARD.md)** | **Nonce Guard:** what an EVM nonce is, pending queues, Wait / Speed Up / Cancel |
| **[ERROR-SYSTEM.md](./ERROR-SYSTEM.md)** | **Error System:** inspect → classify → present → stamp → **Logs** (Log / Errors / Alerts / Connections) |
| **[INTERNAL-DEX.md](./INTERNAL-DEX.md)** | **Internal DEX:** LiFi Worker quotes, atomic 45 bps in the same tx, disabled 0x/V2/V3 fallbacks |
| **[Chrome-Web-Store-path.md](./Chrome-Web-Store-path.md)** | Short operator checklist: verify → rebuild zip → screenshots → listing → submit |
| **[CHROME-WEB-STORE-READINESS.md](./CHROME-WEB-STORE-READINESS.md)** | Operator gap analysis for CWS submission / approval (readiness snapshot **0.11.698**; not for reviewers) |
| **[LOADS.md](./LOADS.md)** | Network loads: comprehensive ping / HTTPS / RPC counts (idle + trading + new EVM nets) |
| **[BUGS-AND-FIXES.md](./BUGS-AND-FIXES.md)** | Known bugs vs by-design vs fixed (historical through **0.11.159**; live product **0.11.698**) |
| **[allow-list/](./allow-list/)** | Inject + network host lists (~118 apex inject hosts; live LiFi MODE is the **production** Worker, not direct `li.quest`) |
| [DOCUMENTATION.txt](./DOCUMENTATION.txt) | Full user guide |
| [HOW-TO-MULTIPLE-LEDGER-WALLETS.txt](./HOW-TO-MULTIPLE-LEDGER-WALLETS.txt) | Ledger multi-wallet how-to |
| **[TERMS-OF-SERVICE.md](./TERMS-OF-SERVICE.md)** | Terms of Service |
| [STORE-LISTING.txt](./STORE-LISTING.txt) | Operator: dashboard paste kit (listing + permissions) |
| [EXTENSION-README.md](./EXTENSION-README.md) | Install, store package, optional Helius |
| [license-bird-colorful-logo-gradient-vector-28267842.pdf](./license-bird-colorful-logo-gradient-vector-28267842.pdf) | Logo / brand license PDF |

## Architecture (short)

**Live product: 0.11.698.** Folder map and fee classes: **[PRODUCT.md](./PRODUCT.md)**. Full detail: **[ARCHITECTURE.md](./ARCHITECTURE.md)**. Current networks: **[CHAINS.md](./CHAINS.md)**. Error System: **[ERROR-SYSTEM.md](./ERROR-SYSTEM.md)**. Nonce Guard: **[NONCE-GUARD.md](./NONCE-GUARD.md)**. Internal DEX: **[INTERNAL-DEX.md](./INTERNAL-DEX.md)**. Messaging: **[MESSAGING.md](./MESSAGING.md)**. Key protection: **[Key-protection-in-Smart-Wallet.md](./Key-protection-in-Smart-Wallet.md)**. Store readiness: **[CHROME-WEB-STORE-READINESS.md](./CHROME-WEB-STORE-READINESS.md)**.

```text
UI (popup / full page)  ←→  Service worker  ←→  Offscreen signer
        │                         │
        │                         ├── dApp approve / trust
        │                         ├── sol/evm RPC proxy (SW)
        ▼                         │
  chrome.storage + vault          │
  chain-registry + rpc-gateway ───┘  (shared UI + SW; provider scoring)
  tx-intent + transaction-manager (lifecycle + multi-RPC confirm)
  portfolio / price / history / swap / bridge managers
  sw-events + state-coordinator
  ui/ui-logo.js (product mark) + ui-theme tokens (dark/light)
                                  │
                                  ▼
              Allowlisted page inject (Wallet Standard + EIP-1193)
                                  │
                                  ▼
              Public multi-RPC · Jupiter lite-api · production LiFi Worker · CoinGecko · optional Helius · WC
```

1. **UI** — balances, send, swap, bridge, settings, Ledger HID (`app.js` + managers + light/dark theme)  
2. **RPC stack** — `chain-registry` · `cache-coordinator` · `rpc-gateway` · `rpc-manager` (shared lists; sequential failover; SW proxy; host scoring)  
3. **Transaction pipeline** — intent → lifecycle states → multi-RPC confirm → events  
4. **Service worker** — dApp messages, auto-lock, signer cache, Solana/EVM RPC proxy  
5. **Page inject** — only allowlisted dApps (`inject-allowlist.js`; see [allow-list/](./allow-list/))  
6. **Software wallets** — seeds always encrypted at rest; plain keys only for sign/approve  
7. **Ledger** — keys on device; public addresses only in extension  
8. **Live data** — idle-first free WebSockets + HTTP fallbacks; activity events refresh portfolio lightly  
9. **Fees** — Jupiter swap **0.45%** · LiFi EVM swap currently **0.70%** combined (0.45% + LI.FI 0.25%) · LiFi EVM-source bridge currently **1.10%** (0.85% + 0.25%) · none on Send / external DEX  
10. **Logo** — in-app product mark centralized in `ui/ui-logo.js` (toolbar icons remain separate)

**Deep dive:** [ARCHITECTURE.md](./ARCHITECTURE.md) · [CHAINS.md](./CHAINS.md) · [ERROR-SYSTEM.md](./ERROR-SYSTEM.md) · [INTERNAL-DEX.md](./INTERNAL-DEX.md) · user-facing short form: **DOCUMENTATION §6**.

## Quick map (DOCUMENTATION.txt)

| Topic | Section |
|-------|---------|
| What it is / chains | §1 |
| Install / update | §2–3 |
| Create / addresses | §4–5 |
| How the wallet works | **§6** |
| Home, PnL, prices | §7 |
| Holdings | §8 |
| Send / Receive / **Buy (Onramper)** / History | **§9** (Buy = §9.2.1) |
| Helius / custom Solana RPC | **§9.3 + §15.2** |
| In-wallet Swap | §10 |
| Bridge | §11 |
| Accounts | §12 |
| Ledger | §13 |
| WalletConnect & dApps (allowlist) | **§14** |
| Password model + RPC | **§15** |
| Backup / seed reveal | §16 |
| Security model | **§17** |
| Fees | **§18** |
| Load pings + free live feeds | **§19** |
| Troubleshooting / privacy | §20–21 |

## Password model (short)

| Account type | Behavior |
|--------------|----------|
| **All software** (seed / imported) | Rust/WASM owns software-key custody. Seeds remain encrypted at rest. With Password ON, unlock requires the user password; with Password OFF, the encrypted vault uses the wallet's device-wrap flow. While Rust is active there is **no plaintext popup password**. Routine signing stays inside WASM and returns only the signature or signed transaction. Seed removal remains an explicit **✕** action. |
| **Ledger** | Optional global Ledger password; keys stay on device |

Full detail: **DOCUMENTATION §15**, **§17.2**, and [Key-protection-in-Smart-Wallet.md](./Key-protection-in-Smart-Wallet.md).

## Better Solana History (optional Helius)

1. Free key: [dashboard.helius.dev](https://dashboard.helius.dev)  
2. Accounts → **Advanced – RPC** → paste key → Save  
3. **History → Solana → Refresh**  

Uses Helius only on History open/Refresh — not continuous Home pings.

## Fees and third-party costs

| Action | Smart Wallet | Combined service/platform (currently) |
|--------|--------------|----------------------------------------|
| Internal Jupiter swap | 0.45% | **0.45%** |
| Internal LiFi EVM swap | 0.45% | **0.70%** (includes current LI.FI 0.25%) |
| Internal LiFi EVM-source bridge | 0.85% | **1.10%** (includes current LI.FI 0.25%) |
| Send / external DEX or bridge | 0% | **0%** from Smart Wallet |

Gas, DEX impact and relayer costs are separate. LI.FI’s 0.25% is quote-derived and not paid to Smart Wallet.

Full: [FEE-DISCLOSURE.md](./Chrome-extension-store-for-reviewers/FEE-DISCLOSURE.md)

## Contact

[CONTACTS.md](./Chrome-extension-store-for-reviewers/CONTACTS.md)

