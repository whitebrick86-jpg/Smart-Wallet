# Chrome Web Store — optional host permission justification

**Product:** Smart Wallet 0.11.298  
**Manifest keys:** `optional_host_permissions`  
**Patterns:** `https://*/*`, `wss://*/*`

This document is the submission-ready justification. It is **not** packed into the Chrome Store ZIP.

## What is requested

`https://*/*` and `wss://*/*` are declared only as **optional** host permissions. They are **not** in required `host_permissions` and **not** in content-script matches.

## Why the wildcard declaration exists

Users may paste an Advanced / custom RPC endpoint (for example a private Helius or QuickNode URL) whose domain cannot be known before runtime. Chrome’s permissions API requires a wildcard optional-host declaration so the extension can request a runtime-discovered HTTPS or WSS origin.

## What happens at runtime

1. The user explicitly saves an Advanced / custom RPC URL.
2. Smart Wallet calls `chrome.permissions.request` for **only that exact origin** (and matching `wss://` / `ws://` when subscriptions apply).
3. **HTTPS** is used only for JSON-RPC requests to that origin.
4. **WSS** is used only for optional real-time RPC subscriptions to that same host.

## What this is not used for

- General browsing
- Arbitrary page injection
- Cookies
- Tracking
- Advertising
- Sale of personal data

Page-provider inject remains a separate allowlist of production dApp hosts.

## If the user denies the prompt

Built-in RPC providers continue to work. The user can remove the custom endpoint and revoke the optional permission later.

## Localhost

`localhost` and `127.0.0.1` are excluded from the production package.

## Related documents

- `Chrome-extension-store-for-reviewers/HOST-PERMISSIONS.md`
- `Chrome-extension-store-for-reviewers/PRIVACY-POLICY.md`
- `STORE-LISTING.txt`
