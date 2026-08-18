# Smart Wallet — Modules

**Product version:** 0.11.298  
**Last synced:** 2026-08-18  
**Source tree:** desktop `Gladiator-Wallet-0.6.55` (load-unpacked)  

This file lists **runtime modules** in the extension. It is documentation-only.

---

## 1. Network foundation

| Module | File | Role |
|--------|------|------|
| Chain Registry | `chain-registry.js` | Shared `CHAINS` / EVM networks for UI + service worker |
| Cache Coordinator | `cache-coordinator.js` | TTL cache + in-flight dedupe |
| RPC Gateway | `rpc-gateway.js` | Sequential multi-RPC failover, timeouts, scoring hooks |
| RPC Manager | `rpc-manager.js` | UI-facing RPC helpers |

## 2. Coordination

| Module | File | Role |
|--------|------|------|
| SW Events | `sw-events.js` | Cross-surface event bus helpers |
| State Coordinator | `state-coordinator.js` | UI state coordination |
| Manager Bootstrap | `manager-bootstrap.js` | Registers manager adapters after `app.js` |

## 3. Transaction pipeline

| Module | File | Role |
|--------|------|------|
| Tx Intent | `tx-intent.js` | Intent model |
| Transaction Manager | `transaction-manager.js` | Lifecycle + multi-RPC confirm |

## 4. Product managers

| Module | File | Role |
|--------|------|------|
| Portfolio | `portfolio-manager.js` | Holdings / portfolio paint path |
| Price | `price-manager.js` | Price feeds coordination |
| History | `history-manager.js` | On-chain history path |
| Swap | `swap-manager.js` | Internal swap orchestration hooks |
| Bridge | `bridge-manager.js` | Internal bridge orchestration hooks |
| Live Feeds | `live-feeds.js` | Idle-first market WS + Solana mentions |

## 5. Helpers (pure / UI-adjacent)

| Module | File | Role |
|--------|------|------|
| Fee Helpers | `fee-helpers.js` | Smart Wallet swap **0.45%** · bridge **0.85%** · LI.FI service parsed from quote (currently 0.25%) |
| Address Guards | `address-guards.js` | Address validation helpers |
| Clipboard Guard | `clipboard-guard.js` | Paste/clipboard safety on Send |
| History Filter | `history-filter.js` | Fee/noise filter for history UI |
| Onramp | `onramp.js` | Buy / Onramper presentation glue |
| Config | `config.js` | Optional local config |

## 6. Core application & service worker

| Module | File | Role |
|--------|------|------|
| App UI + vault glue | `app.js` | Main UI logic, vault, paint*, panels |
| Service worker | `background.js` | dApp messages, auto-lock, signer cache, RPC proxy |
| Offscreen signer | `offscreen-sign.js` + `offscreen.html` | Isolated signing helper |
| Inject allowlist | `inject-allowlist.js` | dApp host allowlist (shared SW + content) |
| Content script | `content-script.js` | Isolated bridge + inject orchestration |
| Page boot | `page-boot.js` | MAIN-world boot |
| Page EVM lite | `page-evm-lite.js` | MAIN-world EVM shim |
| Injected provider | `injected.js` | Wallet Standard + EIP-1193 |
| Failsafe UI | `ui-failsafe.js` | Boot failure banner |

## 7. UI presentation (no wallet business logic)

| Module | File | Role |
|--------|------|------|
| Design tokens | `ui/ui-theme.css` | Dark + Light tokens |
| UI components | `ui/ui-components.css` | States, a11y, Logo presentation |
| **Logo** | `ui/ui-logo.js` | Product logo FILE/VER + hydrate (`SmartWalletUI.logo`) |
| UI shell | `ui/ui-shell.js` | Empty/loading helpers |
| UI map | `ui/UI-MODULES.md` | Conceptual UI modules |
| Layout CSS | `styles.css` | Component chrome (uses tokens) |
| Markup | `popup.html` / `index.html` | Shared panels |
| Privacy page | `privacy.html` | In-extension privacy HTML |

## 8. Manifest / packaging

| Artifact | Role |
|----------|------|
| `manifest.json` | MV3 permissions, content_scripts, web_accessible |
| `icons/*` | Product logo + toolbar icons + chain/dApp assets |
| `lib/*` | Bundled deps (ethers, Solana, Ledger, WC, …) |
| `tools/*` | Verify / regression scripts (dev) |

## 9. Root JS inventory (extension folder)

- `address-guards.js`
- `app.js`
- `background.js`
- `bridge-manager.js`
- `cache-coordinator.js`
- `chain-registry.js`
- `clipboard-guard.js`
- `config.js`
- `content-script.js`
- `dapp-approve-lifecycle.js`
- `dapp-origin-policy.js`
- `dapp-provider-bridge.js`
- `dapp-security-warn.js`
- `dapp-trust-boundary.js`
- `diag-events.js`
- `diag-severity.js`
- `evm-error-classify.js`
- `evm-nonce-queue.js`
- `evm-revert-decoder.js`
- `evm-swap-providers.js`
- `external-dapp-intent.js`
- `fee-helpers.js`
- `history-filter.js`
- `history-manager.js`
- `inject-allowlist.js`
- `injected.js`
- `lifi-proxy-config.js`
- `lifi-proxy-transport.js`
- `live-feeds.js`
- `logs-console.js`
- `manager-bootstrap.js`
- `offscreen-sign.js`
- `onramp.js`
- `page-boot.js`
- `page-evm-lite.js`
- `portfolio-manager.js`
- `price-manager.js`
- `rpc-gateway.js`
- `rpc-host-log.js`
- `rpc-manager.js`
- `state-coordinator.js`
- `sw-diag-log.js`
- `sw-events.js`
- `sw-request-budget.js`
- `swap-manager.js`
- `swap-outcome.js`
- `swap-preflight.js`
- `token-logo-allowlist.js`
- `transaction-manager.js`
- `tx-error-present.js`
- `tx-intent.js`
- `ui-failsafe.js`
- `vault-security-events.js`

### UI folder

- `ui/theme-boot.js`
- `ui/ui-logo.js`
- `ui/ui-shell.js`
- `ui/ui-components.css`
- `ui/ui-theme.css`

---

## Fees (platform)

| Path | Fee |
|------|-----|
| Internal Jupiter swap | **0.45%** |
| Internal LiFi EVM swap | **0.45%** + current LI.FI **0.25%** = **0.70%** |
| Internal LiFi EVM-source bridge | **0.85%** + current LI.FI **0.25%** = **1.10%** |
| Send / Receive / external DEX / History view | **None** |
