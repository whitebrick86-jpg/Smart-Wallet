# Smart Wallet — Modules

**Product version:** 0.11.698  
**Last synced:** 2026-09-13  
**Source tree:** desktop production folder / React-Wallet experiment (load-unpacked)  
**Sibling backend:** `lifi backend fee distributor` — see [PRODUCT.md](./PRODUCT.md)

This file lists **runtime modules** in the extension. It is documentation-only — inventory style, no source paste.

---

## 1. Boot / load control

| Module | File | Role |
|--------|------|------|
| Feature loader | `feature-loader.js` | Ordered feature/script load after the core shell |
| Page-script boot | `page-script-boot.js` | MAIN-world / page-script boot orchestration |
| Startup control policy | `startup-control-policy.js` | Cold-start / reload gating so panels do not race the vault |
| Panel shared stubs | `panel-shared-stubs.js` | Shared empty/loading stubs used by deferred panels |
| Failsafe UI | `ui-failsafe.js` | Boot failure banner |

## 2. Network foundation

| Module | File | Role |
|--------|------|------|
| Chain Registry | `chain-registry.js` | Shared `CHAINS` / `EVM_NETWORKS` for UI + SW. **12 nets:** Solana, Ethereum, Bitcoin, Polygon, Sui, Robinhood Chain, Base/Ethereum Base, BNB Smart Chain, Arbitrum One, Optimism, Avalanche C-Chain, **Sonic**. Bridge menus from `bridgeableChains()` / `destinationBridgeableChains()` |
| Cache Coordinator | `cache-coordinator.js` | TTL cache + in-flight dedupe |
| RPC Gateway | `rpc-gateway.js` | Sequential multi-RPC failover, timeouts, scoring hooks |
| RPC Manager | `rpc-manager.js` | UI-facing RPC helpers |
| Managed RPC | `rpc-managed-*.js` | Optional managed-RPC policy / client (off by default; Public RPC is the live path) |
| Solana ALT | `solana-alt-*.js` | Address Lookup Table verify/support for Jupiter versioned txs; Worker fallback host is not a general RPC proxy |

## 3. Coordination

| Module | File | Role |
|--------|------|------|
| SW Events | `sw-events.js` | Cross-surface event bus helpers |
| State Coordinator | `state-coordinator.js` | UI state coordination |
| Manager Bootstrap | `manager-bootstrap.js` | Registers manager adapters after `app.js` |

## 4. Transaction pipeline

| Module | File | Role |
|--------|------|------|
| Tx Intent | `tx-intent.js` | Intent model |
| Transaction Manager | `transaction-manager.js` | Lifecycle + multi-RPC confirm |
| Tx evidence record | `tx-evidence-record.js` | Durable evidence / identity for a signed or broadcast tx |
| EVM error classify | `evm-error-classify.js` | Inspect + name errors (`SmartWalletEvmErrors`) |
| Tx error present | `tx-error-present.js` | User copy + lifecycle truth (`SmartWalletTxPresent`) |
| EVM revert decoder | `evm-revert-decoder.js` | Allowlisted revert selectors (swap-outcome only) |
| EVM nonce queue / guard | `evm-nonce-queue.js` | Shared pending-nonce range, replacement fees, Wait / Speed Up / Cancel. **Does not apply to Solana, Bitcoin, or Sui.** |

Deep dive: **[ERROR-SYSTEM.md](./ERROR-SYSTEM.md)**. Nonce Guard: **[NONCE-GUARD.md](./NONCE-GUARD.md)**.

## 5. Product managers and views

| Module | File | Role |
|--------|------|------|
| Portfolio | `portfolio-manager.js` | Holdings / portfolio paint path |
| Holdings | `holdings-*.js` | Holdings list / token-detail views |
| Price | `price-manager.js` | Price feeds coordination |
| History | `history-manager.js` · `history-*.js` | On-chain history path + History view |
| Messaging view | `messaging-view.js` | Inbox / Messaging UI (see [MESSAGING.md](./MESSAGING.md)) |
| Swap | `swap-manager.js` · `swap-view.js` | Internal swap orchestration + Swap panel |
| Swap execution | `*-execution-controller.js` (swap) | Sign-once / broadcast / confirm for internal swap |
| EVM swap providers | `evm-swap-providers.js` | LiFi-first Internal DEX quotes via the **production** Worker. 0x / V2 / V3 stay listed and **disabled**. |
| Swap preflight | `swap-preflight.js` | Short-lived gas / sim / allowance / rent caches |
| Swap outcome | `swap-outcome.js` | Internal DEX state machine + codes |
| Bridge | `bridge-manager.js` · `bridge-view.js` | Internal bridge orchestration + Bridge panel |
| Bridge execution | `*-execution-controller.js` (bridge) | Sign-once / broadcast / confirm for internal bridge |
| Live Feeds | `live-feeds.js` | Idle-first market WS + Solana mentions |

Deep dive: **[INTERNAL-DEX.md](./INTERNAL-DEX.md)**.

## 6. Fee policy (do not flatten)

| Module | File | Role |
|--------|------|------|
| Fee Helpers | `fee-helpers.js` | Smart Wallet swap **45 bps (0.45%)** · bridge **85 bps (0.85%)** · LI.FI service parsed from quote (currently **25 bps / 0.25%**). Combined **0.70% / 1.10%** is display-only. |
| Platform fee policy | `platform-fee-policy.js` | Best-effort platform fees: never block an otherwise-safe tx solely because fee insert failed; never sign malformed / misdirected / unverifiable fee payloads |
| Jupiter atomic fee policy | `jupiter-atomic-fee-policy.js` | Jupiter may rebuild a fresh fee-free quote and sign **only** that rebuild. LiFi has a narrow fee-unavailable carve-out; no client fee-free rebuild yet |

## 7. Session + sign lifecycles

| Module | File | Role |
|--------|------|------|
| Software session lifecycle | software session module | Unlock / lock / auto-lock for software wallets |
| Software sign lifecycle | software sign module | JIT sign inside Rust/WASM; no plaintext popup password while Rust is active |
| Ledger session lifecycle | ledger session module | Device connect / Link EVM / disconnect |
| Ledger sign lifecycle | ledger sign module | HID sign on device; keys never enter the software vault |
| Rust session resume | `rust-session-resume.js` | Session-only resume envelope while unlocked; no plaintext JS password retained when Rust is active |

Key protection: **[Key-protection-in-Smart-Wallet.md](./Key-protection-in-Smart-Wallet.md)**.

## 8. Helpers (pure / UI-adjacent)

| Module | File | Role |
|--------|------|------|
| LiFi proxy config | `lifi-proxy-config.js` | Live MODE **production** Worker URL. Staging URL is an explicit developer option. **No API key.** |
| LiFi proxy transport | `lifi-proxy-transport.js` | Extension quote/routes/step client for the Worker |
| Address Guards | `address-guards.js` | Address validation helpers |
| Clipboard Guard | `clipboard-guard.js` | Paste/clipboard safety on Send (`clipboardRead` is **optional**) |
| History Filter | `history-filter.js` | Fee/noise filter for history UI |
| Onramp | `onramp.js` | Buy / Onramper presentation glue |
| Config | `config.js` | Optional local config |
| Request budget | `sw-request-budget.js` | Per-screen RPC/API caps (critical send/sign always allowed) |
| Diag Logs | `sw-diag-log.js` · `diag-events.js` | Privacy-safe Settings → Logs (local only) |

## 9. Core application & service worker

| Module | File | Role |
|--------|------|------|
| App UI + vault glue | `app.js` | Main UI logic, vault, paint*, panels |
| Service worker | `background.js` | dApp messages, auto-lock, signer cache, RPC proxy |
| Offscreen signer | `offscreen-sign.js` + `offscreen.html` | Isolated signing helper |
| Inject allowlist | `inject-allowlist.js` | dApp host allowlist (shared SW + content) — **~118** apex hosts |
| dApp approve lifecycle | `dapp-approve-lifecycle.js` | SW approve request state |
| dApp provider bridge | `dapp-provider-bridge.js` | Isolated-world provider result replay / ack |
| Content script | `content-script.js` | Isolated bridge + inject orchestration |
| Page boot | `page-boot.js` | MAIN-world boot |
| Page EVM lite | `page-evm-lite.js` | MAIN-world EVM shim |
| Injected provider | `injected.js` | Wallet Standard + EIP-1193 |

## 10. UI presentation (no wallet business logic)

| Module | File | Role |
|--------|------|------|
| Design tokens | `ui/ui-theme.css` | Dark + Light tokens |
| UI components | `ui/ui-components.css` | States, a11y, Logo presentation |
| **Logo** | `ui/ui-logo.js` | Product logo FILE/VER + hydrate (`SmartWalletUI.logo`) |
| Theme boot | `ui/theme-boot.js` | Theme apply before first paint |
| UI shell | `ui/ui-shell.js` | Empty/loading helpers |
| UI map | `ui/UI-MODULES.md` | Conceptual UI modules |
| Layout CSS | `styles.css` | Component chrome (uses tokens) |
| Markup | `popup.html` / `index.html` | Shared panels |
| Privacy page | `privacy.html` | In-extension privacy HTML |

## 11. Manifest / packaging

| Artifact | Role |
|----------|------|
| `manifest.json` | MV3 permissions, content_scripts, web_accessible. Named `host_permissions` **~360+** after trim; `https://*/*` + `wss://*/*` optional for Custom RPC; `clipboardRead` optional |
| `icons/*` | Product logo + toolbar icons + chain/dApp assets |
| `lib/*` | Bundled deps (ethers, Solana, Ledger, WC, …) |
| `tools/*` | Verify / regression scripts (dev; not in store zip) |

## 12. Root JS inventory (extension folder, 0.11.698)

Documented names only — this repo does **not** publish source.

- `address-guards.js`
- `app.js`
- `background.js`
- `bridge-manager.js`
- `bridge-view.js`
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
- `feature-loader.js`
- `fee-helpers.js`
- `history-filter.js`
- `history-manager.js`
- `history-*.js`
- `holdings-*.js`
- `inject-allowlist.js`
- `injected.js`
- `jupiter-atomic-fee-policy.js`
- `lifi-proxy-config.js`
- `lifi-proxy-transport.js`
- `live-feeds.js`
- `logs-console.js`
- `manager-bootstrap.js`
- `messaging-view.js`
- `offscreen-sign.js`
- `onramp.js`
- `page-boot.js`
- `page-evm-lite.js`
- `page-script-boot.js`
- `panel-shared-stubs.js`
- `platform-fee-policy.js`
- `portfolio-manager.js`
- `price-manager.js`
- `rpc-gateway.js`
- `rpc-host-log.js`
- `rpc-managed-*.js`
- `rpc-manager.js`
- `rust-session-resume.js`
- `solana-alt-*.js`
- `startup-control-policy.js`
- `state-coordinator.js`
- `sw-diag-log.js`
- `sw-events.js`
- `sw-request-budget.js`
- `swap-manager.js`
- `swap-outcome.js`
- `swap-preflight.js`
- `swap-view.js`
- `token-logo-allowlist.js`
- `transaction-manager.js`
- `tx-error-present.js`
- `tx-evidence-record.js`
- `tx-intent.js`
- `ui-failsafe.js`
- `vault-security-events.js`
- software / ledger session + sign lifecycle modules
- `*-execution-controller.js`

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
| Internal Jupiter swap | **0.45%** (best-effort; may rebuild a fresh fee-free quote) |
| Internal LiFi EVM swap | **0.45%** + current LI.FI **0.25%** = **0.70%** display |
| Internal LiFi EVM-source bridge | **0.85%** + current LI.FI **0.25%** = **1.10%** display |
| Send / Receive / external DEX / History view | **None** |

Best-effort policy: never block solely because fee insert failed if the route is otherwise safe; never sign malformed / misdirected / unverifiable fee payloads. LiFi: narrow fee-unavailable carve-out; no client fee-free rebuild yet.
