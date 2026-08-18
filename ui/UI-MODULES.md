# Smart Wallet — UI module map

**Scope:** **UI-only.** Presentation organization, layout, states, and reusable visual modules.  
Business logic stays in `app.js` + managers + SW.

**Do not change while working under this map:** signing, Ledger, transactions, RPCs, balances, swaps, bridges, approvals, fees, storage, encryption, accounts, dApp/provider, WalletConnect, or network behavior. If a UI bug needs those layers, document it as an out-of-scope functional issue instead of patching them.

**Stack:** Vanilla HTML / CSS / JS (no React/Vue). Popup and full wallet share the same markup (`popup.html` ≈ `index.html`).

This map documents **conceptual UI modules** and where they already live. Prefer extending these hooks over inventing parallel UI systems.

---

## Architecture (presentation)

```text
HTML panels (index.html / popup.html)
    ↓  classes + data-go / data-panel
ui/ui-theme.css          ← shared design tokens (Dark + Light)
styles.css               ← component styles using var(--token)
ui/ui-components.css     ← shared UI states / a11y / layout safety / Logo
ui/ui-logo.js            ← product Logo (asset path, version, sizes, hydrate)
ui/ui-shell.js           ← presentation helpers only
app.js paint* / go*      ← data + actions (existing; theme toggle unchanged)
managers                 ← unchanged
```

### Theme model

```text
Shared UI components
  → shared layout / behavior
  → shared design tokens (ui/ui-theme.css)
       → Dark Mode  (html[data-theme="dark"] / :root)
       → Light Mode (html[data-theme="light"])
```

- Toggle remains existing Settings control + `applyUiTheme` / early `localStorage` bootstrap.
- **Do not duplicate** components for light mode — only token values change.
- Component-specific light overrides still live in `styles.css` as `html[data-theme="light"] .foo` until migrated to tokens.

---

## Layout

| Module | Existing home | Notes |
|--------|---------------|--------|
| AppShell | `.shell` | Outer chrome |
| TopBar | `header.topbar` | Accounts chip, chain, address |
| BottomDock | `nav.dock` | Home / Send / History / Accounts |
| PageContainer | `main.stage` + `.panel` | One panel active via `is-active` |

---

## Navigation

| Module | IDs / classes | Driven by |
|--------|---------------|-----------|
| AccountSelector | `#brandAccountsBtn`, `#acctDrawer` | app.js drawer paint |
| ChainSelector | `#chainPicker`, `#chainPickerMenu` | app.js chain picker |
| Navigation | `[data-go]`, `.dock-item` | app.js panel routing |

---

## Home

| Module | IDs / classes | Driven by |
|--------|---------------|-----------|
| PortfolioSummary | `#fiatBalance`, `.balance-block` | `paintBalances` |
| PnL | `#balancePnlDelta` | portfolio paint |
| NativePrice / chart | `#balanceNativePrice`, `#balanceNativeChart` | price feeds + paint |
| QuickActions | `.cta-row` Send/Receive/Swap/Bridge | `data-go` only |
| HoldingsList / HoldingRow | holdings list UL | `paintHoldings` / `paintHoldingsNow` (stable rows) |
| Ledger indicator | `#ledgerBalanceTag` | account type |

---

## Accounts

| Module | Panel / IDs |
|--------|-------------|
| AccountsPage | `#panel-activity` |
| AccountDrawer | `#acctDrawer` |
| AccountRow / actions | photon wallet rows in panel-activity |

---

## Transactions

| Module | Panel / IDs |
|--------|-------------|
| Send | `#panel-send`, `#sendStatus` |
| Receive | `#panel-receive` |
| History | `#panel-history`, `#historyStatus` |
| TransactionStatus | `.form-status` per flow |

---

## Swap / Bridge / Tokens / Settings / Ledger

| Module | Panel |
|--------|--------|
| SwapPage | `#panel-swap` |
| BridgePage | `#panel-bridge` |
| TokenDetail | `#panel-token` |
| SettingsPage | `#panel-settings` |
| Ledger approval / wait | `#ledgerSignModal`, `#ledgerWait*` modals |

---

## Shared components (CSS + thin JS)

| Concept | Location |
|---------|----------|
| **Logo** | `ui/ui-logo.js` (`SmartWalletUI.logo`) + `.sw-logo` / `.sw-logo--sm|md|lg` in `ui/ui-components.css`. HTML: `[data-sw-logo]`. **Authoritative:** change `FILE` + `VER` only in `ui/ui-logo.js`. |
| Buttons | `.btn`, `.btn-send`, `.btn-swap`, … in `styles.css` |
| Inputs | form fields in panels |
| Modal shell | `.confirm-modal`, `.confirm-dialog` |
| Toast | `#toast` |
| Loading / empty / error | `ui/ui-components.css` (`.ui-state-*`) + `ui/ui-shell.js` |
| Design tokens | `ui/ui-theme.css` (`:root` + `html[data-theme="light"]`) + residual component chrome in `styles.css` |

### Logo API (`SmartWalletUI.logo`)

| Member | Purpose |
|--------|---------|
| `FILE` | Extension-relative path (`icons/smart-wallet.png`) |
| `VER` | Cache-bust string |
| `pageUrl()` | `./FILE?v=VER` for popup/index HTML |
| `extUrl()` | `chrome.runtime.getURL(FILE)` (content-script inject) |
| `assetUrl()` / `url()` | Best URL for current context (+ `?v=`) |
| `apply(img, variant)` | Set src/classes/size on an `<img>` |
| `hydrate(root)` | Apply to all `[data-sw-logo]` under root |

### Logo usage map (product mark only)

| Surface | Variant | Hook |
|---------|---------|------|
| Favicon | favicon | `<link data-sw-logo="favicon">` + hydrate |
| TopBar brand chip | sm (20) | `.brand-mark.sw-logo--sm[data-sw-logo="sm"]` |
| Accounts drawer | lg (72) | `.sw-logo--lg[data-sw-logo="lg"]` |
| dApp approve default | md (40) | `#dappApproveLogo` (site favicon may replace `src` at runtime) |
| History internal swap icon | — | `productLogoUrl()` in `app.js` |
| dApp approve logo fallbacks | — | `productLogoUrl()` in `app.js` |
| Page inject list logo | — | `content-script.js` via `SmartWalletUI.logo.extUrl()` (ui-logo.js loaded first) |
| Privacy page favicon | favicon | `privacy.html` + `ui/ui-logo.js` |

### Explicitly **not** the product Logo module

| Asset | Why separate |
|-------|----------------|
| `icons/icon16.png` … `icon128.png` | Chrome **toolbar / store** icons (`manifest.action.default_icon`) |
| `icons/solana.png`, `ethereum.png`, … | **Chain** logos |
| `icons/dapps/*` | dApp list icons |
| `icons/usdc.png` | Token asset |

---

## Files in this folder

| File | Role |
|------|------|
| `ui-theme.css` | **Shared design tokens** — Dark + Light (`html[data-theme]`) |
| `ui-components.css` | Shared empty/loading/error/a11y/dock/Logo presentation — uses tokens |
| `ui-logo.js` | Product Logo source of truth + hydrate |
| `ui-shell.js` | Presentation helpers only; never holds secrets or signs |
| `UI-MODULES.md` | This map |

Do **not** move vault, RPC, swap routing, or signing into `ui/`.  
Do **not** fork light-mode HTML or separate light components.
