# Smart Wallet — allow list

Folder for **explicit site / host lists** used (or proposed) instead of global  
`https://*/*` / `wss://*/*` network access.

| File | What it is |
|------|------------|
| [network-hosts.md](./network-hosts.md) | RPC, API, explorer, WC, logo CDN hosts for **`host_permissions`** |
| [host_permissions-draft.txt](./host_permissions-draft.txt) | Copy-paste permission patterns for `manifest.json` |
| [inject-dapp-hosts.md](./inject-dapp-hosts.md) | dApp sites where the wallet **provider injects** (page allowlist) |
| [inject-allowlist-source.js](./inject-allowlist-source.js) | Snapshot of extension `inject-allowlist.js` |

## Two different lists

1. **Network allow list** — extension process may `fetch` / open WebSockets to these hosts (balances, swap, bridge, history, prices).  
2. **Inject allow list** — only these **websites** get content scripts / Wallet Standard / EIP-1193.

Global `https://*/*` is **not** the same as inject-everywhere. Inject is already restricted to the inject list.

## Status

- **Inject list snapshot:** extension **0.11.298** in `inject-allowlist-source.js`. Localhost and `127.0.0.1` are **not** production inject hosts.  
- **LiFi network path:** live MODE is production `https://smart-wallet-lifi-proxy.smart-wallet.workers.dev`. The extension does not call `li.quest`.  
- Required `host_permissions` are named RPC / API / explorer / WC hosts.  
- **`optional_host_permissions`**: `https://*/*` and `wss://*/*` are optional only. The wallet requests the **exact user-selected** Advanced / custom RPC origin at runtime.  
- Content scripts load: `inject-allowlist.js` → `dapp-provider-bridge.js` → `content-script.js`.  
- Reload the extension after updating for Chrome to re-read the manifest.

## Related

- [HOST-PERMISSIONS.md](../Chrome-extension-store-for-reviewers/HOST-PERMISSIONS.md) — why broad network was used  
- [CHROME-WEB-STORE-READINESS.md](../CHROME-WEB-STORE-READINESS.md) — store submit notes  
