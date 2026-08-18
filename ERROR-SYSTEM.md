# Error System

**Product:** Smart Wallet (Chrome / Opera MV3 extension)  
**Error-system snapshot:** **0.11.233** (this document’s Logs / Warnings / Connections body)  
**Live product:** **0.11.257** — see [PRODUCT.md](./PRODUCT.md)  
**Original architecture snapshot:** **0.11.159** (inspect → classify → present → stamp)  
**Later additions (not in 0.11.159):** Logs store **0.11.145**; in-wallet Log / Errors / Alerts UI **0.11.191–0.11.197**; **Connections** tab **0.11.204**; newest-first **0.11.205**; Connections-only rows **0.11.206**; tab-scoped Clear **0.11.208**; SW-serialized log writes + `originalErr` isolation **0.11.209**; external DEX swap failures in Logs **0.11.210**; external DEX failure **reason** on the Swap row **0.11.211**; external dApp **intent** (not hostname=swap) + RPC host-failure rows **0.11.212**; expanded structured diagnostic catalog + 500/1000 cap + recovery **0.11.213**; four-level severity (critical/error/warning/info) **0.11.214**; muted four-color palette **0.11.215**; dApp **Warnings** tab **0.11.216**; vault / seed-reveal / unrecognized-outgoing Warnings **0.11.217**; muted palette + red Connections disconnect **0.11.218**; Home critical-warning badge **0.11.219**; badge copy **review logs** **0.11.220**; Error System docs: selected-tab Clear, plaintext-boundary claim, Ledger seed-warning guidance **0.11.220**; Logs UI chrome polish **0.11.221**; light-mode Logs canvas fill + banner removed **0.11.222**; Logs leftover `#3d3e46`, dock/home/send/history/accounts untouched **0.11.223**; Logs top-chip hover stays Home plates **0.11.224**; Logs gold tab / `#3d4d60` box border restored **0.11.225**; Logs rounded `#121a24` shell **0.11.226**; leftover `#121a24` / inner outline `#3d3e46` **0.11.227**; vault-watcher false-positive fix **0.11.228**; scoped vault-write protocol **0.11.229**; unauthorized vault warning no longer pauses signing **0.11.230**; owner-write stamp so only non-owner vault changes warn **0.11.231**; critical tx mismatches ask to proceed **0.11.232**; Logs Settings switches persist across popup close **0.11.233**; Error System §9.3.1 documents **Hide routine success**; Error System §9.7 **Warnings that may be triggered by owner** **2026-08-16**  
**Last updated:** 2026-08-16  
**Repository:** Documentation only — extension source is **not** published here.

This is the account of the **in-wallet Error System**: how a raw RPC, Ledger, provider, or contract failure becomes a named code, a single honest user sentence, a privacy-safe diagnostic, and (on the verified paths below) a local Logs row.

It is **not** a crash reporter, **not** a `console.log` interceptor, and **not** uploaded anywhere.

---

## 1. What the Error System is

The Error System is a **layered pipeline**. These paths are **verified** to go through it today: **Send**, **internal Swap**, **external dApp** send/sign/message-signing failures, **RPC host** failover / exhaustion, **internal Bridge**, and **Ledger** connect / broadcast-proof. Other subsystems are listed in §9.2 as not wired. This document does **not** claim every catch in the wallet uses the shared system.

```text
Raw failure (ethers / JSON-RPC / Ledger / provider / revert bytes)
        │
        ▼
  1. Inspect     walk nested objects + embedded JSON; extract facts; redact secrets
        │
        ▼
  2. Classify    one canonical code (sticky vs RPC vs delivery vs lifecycle)
        │
        ▼
  3. Present     one user message + one lifecycle truth sentence
        │
        ▼
  4. Stamp       mark already-presented so later catches do not rewrite or double-append
        │
        ▼
  5. Surface     toast / send-form / swap-outcome / Logs (local only)
```

**Inspect and classify do not talk to the network.** Sequential RPC callers decide failover. The Error System only **names** what already happened and **stops** callers from treating infrastructure failures as “not enough coins,” or a submitted hash as “failed.”

---

## 2. Modules (runtime)

| Module | Global | Role |
|--------|--------|------|
| `evm-error-classify.js` | `SmartWalletEvmErrors` | Inspect + classify. No user copy. No secrets. |
| `tx-error-present.js` | `SmartWalletTxPresent` | User-facing wording + lifecycle truth + stamp. |
| `evm-revert-decoder.js` | (consumed by swap-outcome) | Decode allowlisted ERC-20 / router revert selectors. Does **not** render copy. |
| `swap-outcome.js` | `SmartWalletSwapOutcome` | Internal-DEX state machine + swap-specific codes + DevTools journal. |
| `sw-diag-log.js` | `SmartWalletDiagLog` | Privacy-safe **Logs** page. Local `chrome.storage.local`. Never uploaded. |

`app.js` keeps **thin wrappers** (`presentSendCaughtError`, `friendlySendError`, …) that inject `activeChain(STATE)` and call `SmartWalletTxPresent`. Extraction Batch 1 (**0.11.154**) moved the remaining send presentation body out of `app.js` without changing classification or wording.

---

## 3. Design rules (do not break)

1. **Classifier names; caller failovers.** Never start a new RPC from the presenter.
2. **One request, one healthy host.** Sequential only. A deterministic account reject **stops** the host walk.
3. **Sticky codes stay sticky.** `PENDING_BALANCE_RESERVED`, `INSUFFICIENT_CONFIRMED_BALANCE`, user-reject, nonce/revert codes are not remapped because a later host returned `429` or a truncated string.
4. **Hash present = submitted.** Lookup miss / empty receipt / RPC lag is `CONFIRMATION_DELAYED`, never automatic `FAILED`.
5. **Ambiguous timeout after a signed broadcast = `BROADCAST_UNCERTAIN`.** Do not re-sign. Do not invent a second payload. Check History / explorer before retry.
6. **One lifecycle sentence.** Later catches that see `isPresentedTxError` **pass through**.
7. **No raw dump.** No signed raw, calldata, full addresses, seeds, passwords, API keys, or stack traces in the user message or Logs.
8. **Native asset, not “gas,” on non-EVM.** Solana / Bitcoin / Sui wording says “network fee,” never EVM gas.
9. **Proof beats regex.** Numeric `have` / `want` / `queuedCost` / `txCost` from nested ethers objects beat flattened `message` text.
10. **HTTP 401/403/404/429/5xx and DNS are RPC class**, never a native-balance shortfall.

---

## 4. Inspection (`inspectError`)

Shipped **0.11.141**, completed **0.11.144**.

Ethers wraps Geth/Nethermind rejects as `could not coalesce error` with the real payload nested under `info.error.message` / `info.error.data` / `cause` / `error={…}` JSON inside a string. Flattening `.message` first **drops** queued/tx/have/want and used to produce `UNKNOWN` plus a raw dump.

`inspectError` walks (depth ≤ 8, cycle-safe):

| Source | What is taken |
|--------|----------------|
| `message`, `shortMessage`, `reason`, `details` | Text + embedded `error={…}` JSON |
| `cause`, `error`, `info`, `data`, `payload` | Nested objects |
| `info.error`, `info.error.data`, `data.error` | Typical ethers coalesce shape |
| Numeric / named fields | `have`, `balance`, `want`, `required`, `queued` / `queuedCost`, `txCost`, `overshoot`, `nonce`, `replacementFee` |
| `code` | ethers `INSUFFICIENT_FUNDS` **and** JSON-RPC numeric codes |
| `status` / `httpStatus` / `HTTP 429` in text | HTTP class |

**Skipped walk keys** (never inspected, never copied into facts): `rawTransaction`, `signedTransaction`, `signedRaw`, `rawTx`, `calldata`, `privateKey`, `secretKey`, `mnemonic`, `seed`, `password`, `authToken`, `apiKey`, `authorization`, `stack`, `params`.

Long hex (`0x` + ≥20 hex chars) is treated as opaque, not an amount. Amounts are parsed as wei (integer or decimal → 18-dec). Facts are **sanitized** before any UI or Logs use.

**0.11.144 rule:** parse nested `message` / `reason` / `info.error.message` **before** any redact, then classify the **original error object** (Ledger must not pass only `bcErr.message`).

---

## 5. Canonical codes

`SmartWalletEvmErrors.CODES` (aliases normalize to the right-hand name):

### 5.1 Account / funds (sticky)

| Code | Meaning | User idea |
|------|---------|-----------|
| **`PENDING_BALANCE_RESERVED`** | Confirmed native **covers this tx**, but a **queued / pending** cost makes the combined requirement fail | “Another pending tx is reserving your POL / ETH / BNB.” |
| **`INSUFFICIENT_CONFIRMED_BALANCE`** | Confirmed native does **not** cover this tx (amount + network fee). Aliases: `INSUFFICIENT_NATIVE_FUNDS`, `INSUFFICIENT_GAS` | “Not enough ETH / POL / SOL …” |
| `SIMULATION_REVERT` | `eth_call` / estimate would revert | Would revert |
| `TOKEN_CONTRACT_REVERT` | Token / allowance / transferFrom reject | Token rejected this tx |
| `NONCE_TOO_LOW` / `NONCE_TOO_HIGH` | Nonce already used / too far ahead | Check History; wait |
| `REPLACEMENT_UNDERPRICED` | Replacement fee below pending | Raise fee or wait |
| `TRANSACTION_ALREADY_KNOWN` | Mempool already has this exact payload | Submitted — do not send again |
| `USER_REJECTED` | User / Ledger cancelled | Cancelled; nothing submitted |

**This pair is the heart of the Error System.** Live Polygon Ledger (0.11.139–0.11.144) had ~274 POL confirmed, this send needed ~264 POL, and the RPC said `queued cost: 273.8` vs `tx cost: 263.8`. A regex on `insufficient funds for gas * price + value` used to show **Not enough POL**. That is **false**. The wallet had the coins; another pending tx had reserved them.

`PENDING_BALANCE_RESERVED` is raised only when:

- numeric `have` (or proof balance) **≥ `txCost`**, **and**
- `queuedCost` **> `txCost`**, **and**
- the RPC text is a deterministic insufficient-funds shape (not HTTP/DNS/429).

If confirmed balance does **not** cover `txCost`, the code is `INSUFFICIENT_CONFIRMED_BALANCE`.

### 5.2 RPC / infrastructure (not a balance bug)

| Code | Typical evidence |
|------|------------------|
| `RPC_AUTH_REQUIRED` | HTTP 401 / 403 / API key |
| `RPC_RATE_LIMITED` | HTTP 429 / Retry-After |
| `RPC_UNAVAILABLE` | timeout, fetch failed, 502/503/521 |
| `RPC_DNS_FAILURE` | `ENOTFOUND` / name not resolved |
| `RPC_ENDPOINT_NOT_FOUND` | HTTP 404 |
| `RPC_SERVER_ERROR` | HTTP 5xx (non-unavailable) |

**Permanent RPC** (do not keep hitting the same host): `RPC_AUTH_REQUIRED`, `RPC_DNS_FAILURE`, `RPC_ENDPOINT_NOT_FOUND`. Long cooldown (0.11.137).

### 5.3 Broadcast / confirm / delivery

| Code | Meaning |
|------|---------|
| `BROADCAST_REJECTED` | Host explicitly rejected; **no** hash |
| `BROADCAST_UNCERTAIN` | Broadcast attempted; timeout / missing response; **no reliable hash** |
| `CONFIRMATION_DELAYED` | Hash exists; receipt not in yet (alias: `CONFIRMATION_TIMEOUT`) |
| `PROVIDER_REQUEST_TIMEOUT` | Popup ↔ service worker port closed (`message port closed`) |

### 5.4 Ledger (pre-broadcast, sticky)

| Code | Meaning |
|------|---------|
| `LEDGER_WRONG_APP` | Ethereum app not open |
| `LEDGER_DISCONNECTED` | HID gone |
| `LEDGER_BLIND_SIGN_REQUIRED` | Blind signing off |
| `LEDGER_ERROR` | Device rejected / other APDU |

### 5.5 Fallback

| Code | Meaning |
|------|---------|
| `UNKNOWN` | Could not name it. Presenter still must **not** dump raw RPC. |

---

## 6. Presentation (`SmartWalletTxPresent`)

Shipped **0.11.140**, extracted fully **0.11.154**.

### 6.1 Lifecycle truth (always one of these)

Computed from flags, not from the raw string:

| Condition | Truth |
|-----------|--------|
| Valid hash / broadcast accepted | **Transaction submitted.** (or **… confirmation is delayed.**) |
| Broadcast attempted + ambiguous timeout, no hash | **Submission status is uncertain. Check the transaction hash before retrying.** |
| Signed + explicit reject | **Transaction was signed but rejected before submission.** |
| Signed, not attempted | **Transaction was signed but not submitted.** |
| Not signed | **Transaction was not signed or submitted.** |

A valid identity is chain-specific (`isValidSendTxIdentity` / presenter `validHashPresent`):

| Family | What counts as a hash |
|--------|------------------------|
| EVM | `0x` + 64 hex |
| Solana | base58, typically ≥ 64, **not** `0x` |
| Bitcoin | 64 hex (optional `0x`) |
| Sui | digest, length ≥ 40 |

Forcing every chain through the EVM regex was the **0.11.36** “no hash found” bug after a successful Solana/BTC/Sui send.

### 6.2 Native labels

Presenter maps chain id / name to the **gas coin the user holds**:

| Chain | Symbol shown |
|-------|----------------|
| Ethereum / Base / Robinhood ETH (4663) | ETH |
| BNB Smart Chain | BNB |
| Polygon | POL |
| Solana | SOL |
| Bitcoin | BTC |
| Sui | SUI |

`PENDING_BALANCE_RESERVED` example (Polygon, signed):

> Send blocked — another pending Polygon transaction is reserving your POL. Transaction was signed but rejected before submission. Wait for the pending transaction to confirm or replace/cancel it.

Solana rent residual uses a dedicated sentence (minimum rent reserve ~0.00089 SOL + fees) — protocol, not an app bug.

### 6.3 Stamp contract (`stampPresented` / `presentCaughtError`)

The stamped Error carries:

| Field | Role |
|-------|------|
| `isPresentedTxError` | Later catches must not re-present |
| `userMessage` | Final copy |
| `classification` / `swCode` / `swRootCode` | Canonical code |
| `signed` / `swSigned` | Ledger or software actually signed |
| `broadcastAttempted` | `eth_sendRawTransaction` (or equivalent) was called |
| `broadcastAccepted` / `txHashPresent` | Network returned an identity |
| `submissionState` | `submitted` · `signed_rejected` · `signed_not_submitted` · `not_signed` |
| `swFacts` / `swProof` | Sanitized numbers only |
| `originalErr` | Nested original for Ledger `decideLedgerBroadcastError` |

If a catch sees `isPresentedTxError && userMessage`, it **returns that object**. This stopped 0.11.143/144 from appending a second “was not signed” sentence onto an already-correct pending-balance message.

---

## 7. Who calls what

```text
Send form / token / native / preflight estimateGas
    → presentCaughtError / presentSendCaughtError
        → SmartWalletTxPresent.present
            → SmartWalletEvmErrors.classify  (EVM)
            → classifyNonEvm                 (SOL / BTC / Sui)

Ledger sequential broadcast
    → decideLedgerBroadcastError(original bcErr)
        → first PENDING_BALANCE_RESERVED / INSUFFICIENT_… / nonce / revert
          STOPS the host loop (0.11.142). Same signed raw is not sent to hosts 1–7.

Software EVM sign-once (0.11.150)
    → signAndBroadcastEvmSoftwareOnce
        → deterministic reject stops; timeout rebroadcasts THE SAME raw only.

Internal Swap
    → SmartWalletSwapOutcome (quote / sim / allowance / revert codes)
    → presenter for broadcast / funds / RPC class
    → later-receipt fee machine does not invent errors (see INTERNAL-DEX.md)

Internal Bridge
    → same presenter on send/fee legs; quiet status poll does not surface as a new “failed send”

dApp / injected provider
    → provider errors for unknown chain / stale accounts (0.11.152)
    → durable result tracker delivers hash/error once (0.11.150); never re-signs
```

**Pre-sign `estimateGas` (0.11.143)** must go through the presenter **before** the Nano prompt. An intrinsic-cost reject with nested queued/tx is `PENDING_BALANCE_RESERVED` and must never reach the device as a sign request, and must never appear as raw `info={transaction=…}`.

---

## 8. Swap-specific layer (`swap-outcome.js`)

Internal DEX has its **own** state machine and codes (quote / route / allowance / fee settlement). It **reuses** the Error System for funds, RPC, broadcast, and lifecycle. It does **not** invent a second presenter.

Representative swap codes (not a second classification universe — they map onto the same user rules):

| Family | Examples |
|--------|----------|
| Provider | `PROVIDER_RATE_LIMITED`, `PROVIDER_TIMEOUT`, `PROVIDER_UNAVAILABLE`, `NO_ROUTE`, `QUOTE_EXPIRED` |
| Wallet | `INSUFFICIENT_BALANCE`, `INSUFFICIENT_GAS`, `PENDING_BALANCE_RESERVED`, `SIGNATURE_REJECTED` |
| Route | `NO_CONNECTOR_PATH`, `ALL_ROUTES_SIM_FAILED`, `CANDIDATE_BUDGET_EXHAUSTED` |
| On-chain | `SIMULATION_REVERTED`, `ONCHAIN_REVERT`, `MIN_OUTPUT_EXCEEDED`, `DEADLINE_EXPIRED` |
| Fee | `PLATFORM_FEE_PENDING`, `PLATFORM_FEE_TRANSFER_FAILED` — fee **after** main-swap confirm only |

`evm-revert-decoder.js` ranks evidence from **allowlisted** selectors only (OpenZeppelin ERC-20 custom errors, verified router `Error(string)` / V3 short codes). Untrusted 4-byte signatures stay `UNKNOWN_CONTRACT_REVERT`. No calldata in the user message.

Correction caps (do not retry-storm): 1 diagnostic attempt, 1 approval correction, 2 quote refreshes, 2 provider replacements, 2 route replacements.

Full quote/execute architecture: **[INTERNAL-DEX.md](./INTERNAL-DEX.md)**.

---

## 9. Logs

Settings → **Logs** → **Open** is the in-wallet error console. It is part of the Error System (step 5 — Surface). It is **not** a crash reporter, **not** a `console.log` interceptor, and **not** uploaded anywhere.

Open stays **inside the wallet** (`#panel-logs`). There is no separate browser window.

### 9.1 Features

| Feature | Behavior |
|---------|----------|
| **Where** | Settings → Logs → **Open**. Back link returns to Settings. |
| **Storage** | Local only — `chrome.storage.local` key `smart_wallet_diag_logs` |
| **Cap** | Default **500** events, optional Advanced **1,000**. FIFO (oldest dropped first). Shipped 0.11.213 (was 300 at 0.11.193, 100 at 0.11.145). |
| **Same store, five views** | **Log**, **Errors**, **Alerts**, **Warnings**, and **Connections** do **not** record separately. One bag; each tab only filters / formats it. A row may appear in more than one tab; it is persisted once. |
| **Pills** | `errors N` · `warnings N` · `events N` — `errors` is **critical + error** |
| **Stats line** | “Errors now · Warnings · Events logged · last 500 on this device” |
| **Dedupe** | The same event inside **8 seconds** collapses to one row with a repeat count (`xN`) |
| **Blocking** | Fire-and-forget. A log write **never** awaits on send / swap / bridge / sign / broadcast / confirm. Failed storage writes are swallowed. |
| **Upload** | **None.** No telemetry, no crash dump, no network. |
| **Clear** | **Clear logs** deletes the **selected tab’s logs** (the tab you are viewing). It does **not** wipe the entire store or the other tabs. Confirm copy names the tab. Shared rows that also appear on another tab leave those views too, because there is one bag. |
| **Copy** | **Copy selected log** copies the clicked row. There is no copy-all. |
| **Home badge** | Total balance shows a red-dot count of stored **critical** Warnings-tab rows plus **review logs**. Hidden at 0. Click opens Logs → Warnings. |
| **Filters** | Severity / subsystem / chain dropdowns were removed (0.11.192). Use the five tabs. |

### 9.1.1 The five tabs — what each one shows

Nothing is written *to* a tab. Send / Swap / Bridge / Ledger / DApp write **one row** into the shared store. Opening a tab only chooses which of those rows you see.

| Tab | What it shows | What it hides | Layout |
|-----|----------------|---------------|--------|
| **Log** | All non-Connections rows: `critical`, `error`, `warning`, and `info`. | **dApp connect / disconnect** (those live only on Connections). | **Newest first — earliest logs shown last.** One short line per event: **timestamp**, `CRITICAL`/`ERROR`/`WARNING`/`INFO`, subsystem, chain, code, message. Separated by `---`. |
| **Errors** | `critical` and `error` only. | Warnings, info, and Connections rows. | **Newest first — earliest logs shown last.** Two-tone list: **timestamp** + severity + repeat on the left; subsystem · chain and `[code]` detail on the right. Critical rows are red; error rows are orange. Click a row, then **Copy selected log**. |
| **Alerts** | `critical`, `error`, and `warning` (the things that need attention). | Info and Connections rows. | **Newest first — earliest logs shown last.** Blocks: `[timestamp] CRITICAL/ERROR/WARNING code= role=` then Chain / Repeat / Detail. No `---` separators. |
| **Warnings** | dApp origin / approval / review-integrity **and** vault, seed-reveal, plaintext-sink, and unrecognized-outgoing events (by code). Severity colors stay red/orange/yellow/green. | Unrelated send/RPC rows and Connections. | **Newest first.** Same stream layout as Log. Critical mismatches stay red. |
| **Connections** | **dApp connect and disconnect only** (injected sites + WalletConnect). Same store, filtered to `DAPP_CONNECT` / `DAPP_DISCONNECT`. Stored as `info`. | Send / swap / bridge / Ledger rows. | **Newest first — earliest logs shown last.** Each row has a **timestamp**. **Connect is green. Disconnect is red.** |

All five tabs use the same order: the **latest** log is at the top; the **earliest** log is shown last. Every row has a timestamp (`YYYY-MM-DD HH:MM:SS`). A missing `ts` is filled with “now” when the row is stored or painted.

### 9.1.2 Connections tab

Settings → Logs → **Connections** (shipped **0.11.204**).

This tab is the dApp connection history. It is **not** a live list of who is connected right now. It is a chronological log of connect and disconnect events, each with a timestamp.

| Item | Behavior |
|------|----------|
| **What it shows** | Only `DAPP_CONNECT` and `DAPP_DISCONNECT` rows (injected sites + WalletConnect). |
| **What it hides** | Send / swap / bridge / Ledger errors and warnings. |
| **Timestamp** | Every row: `[YYYY-MM-DD HH:MM:SS]`. |
| **Connect** | Green (`--log-info` / `#6eae90`). Line reads `CONNECT` plus the site / WalletConnect name. |
| **Disconnect** | Red (`--log-disconnect` / `#c87a7e`). Line reads `DISCONNECT` plus the site / WalletConnect name. Display color is red so disconnects are easy to scan. The stored row stays `info` (a factual history event, not a failed transaction). If an unexpected disconnect interrupts an active operation, that interrupted operation is a separate warning or error. |
| **Layout** | **Newest first — earliest logs shown last.** Same `---` separators as the Log tab. |
| **Empty** | “No dApp connections yet. Connect or disconnect a site to see it here.” |

**When a Connections row is written**

| Event | Code | Color |
|-------|------|--------|
| First-time inject connect (user approves Uniswap / Jupiter / PancakeSwap / etc.) | `DAPP_CONNECT` | green |
| First time that same site also connects the other chain (EVM after Solana, or the reverse) | `DAPP_CONNECT` | green |
| WalletConnect session approved | `DAPP_CONNECT` | green |
| Site disconnect (`wallet_revokePermissions`, Solana `disconnect`, Settings disconnect, WalletConnect disconnect) | `DAPP_DISCONNECT` | red |

**Not written**

- Reloading a page you already approved (silent rehydrate)
- Signing / swapping on a site that is already connected
- Failed connect prompts the user rejected (that is a warning on Log / Alerts, not a Connections connect)

Those DApp rows appear **only** on **Connections**. They do **not** appear on **Log**, **Errors**, or **Alerts**.

**Privacy:** Connections is **local dApp connection history** on this device (which sites you connected or disconnected). It is **never uploaded**. Clear the Connections tab to delete that history. The Logs banner states this.

**Log tab colors** (four-level, 0.11.214):

| Stored | Label | Color | What those rows are |
|--------|--------|--------|---------------------|
| `critical` | CRITICAL | red `--log-critical` `#c98a8e` | Uncertain fund movement, signed-payload mismatch, vault integrity, or similar rare risk |
| `error` | ERROR | orange `--log-error` `#c4a06a` | Operation failed; wallet knows funds/state remain safe |
| `warning` | WARNING | yellow `--log-warning` `#c4b478` | Degraded, delayed, fallback, or user rejection |
| `info` | INFO | green `--log-info` `#6eae90` | Success, lifecycle, or recovery |

Light-mode Logs title stays gold (same as a clicked Alerts tab), not purple. Secrets stay off the screen — messages are escaped HTML; nothing in this console is a seed, password, key, signed raw, or full calldata.

**Sanitize before persist:** only allowlisted fields are stored (`id`, `ts`, `severity`, `subsystem`, `chain`, `message`, `code`, `repeat`, `fingerprint`). `originalErr` / raw `Error` objects are never written. Recursive JSON key redaction (`password`, `mnemonic`, `seed`, `privateKey`, `signedTransaction`, `calldata`, `apiKey`, …) plus `redactEmbeddedSecrets` on strings. Long hex is treated as opaque.

**One writer:** popup, offscreen, and other pages send `{ type: smart-wallet-diag-record }` to the service worker. Only the SW persists `chrome.storage.local` (serialized `writeTail` queue). Concurrent UI writes cannot overwrite each other.

**Paint:** Log / Errors / Connections rows are built with `textContent` / DOM nodes. Stored messages, site names, chain labels, and codes are never assigned through raw `innerHTML`.

**Clear:** **Clear logs** deletes the **selected tab’s logs**, then immediately repaints all five views. It does not wipe the entire store.

Logs are **not a network load**. See [LOADS.md](./LOADS.md).

Module: `sw-diag-log.js` (`SmartWalletDiagLog`). UI wrappers in `app.js`: `recordWalletDiag` / `recordWalletDiagFromError` / `paintLogs`. Authoritative persist: service worker.

### 9.2 What events it records throughout the wallet

Every stored row is an **intentional structured event** with:

`severity` · `subsystem` · `chain` · `code` · `message` · `ts` · `repeat`

The presenter / classifier supplies the **code** and the **user sentence** when one exists (`swCode` / `userMessage`). User-cancelled / “user rejected” is stored as a **warning**, not an error.

**Wired today** (these wallet paths write a Logs row):

| Subsystem | When a row is written | Typical codes / copy |
|-----------|------------------------|----------------------|
| **Send** | Send form / native or token send catch (`executeSend`) | `PENDING_BALANCE_RESERVED`, `INSUFFICIENT_CONFIRMED_BALANCE`, `NONCE_*`, `REPLACEMENT_UNDERPRICED`, `BROADCAST_REJECTED`, `BROADCAST_UNCERTAIN`, `RPC_*`, `USER_REJECTED` (warning) |
| **Swap** | Internal DEX execute catch | Same funds / RPC / broadcast family, plus swap-outcome codes (`NO_ROUTE`, `SIMULATION_REVERTED`, `MIN_OUTPUT_EXCEEDED`, `SIGNATURE_REJECTED`, …) |
| **DApp** | External site `eth_sendTransaction` / Solana `signTransaction` / `signAndSendTransaction` / `signAllTransactions` fail or user reject | **Action** from verified selector / program evidence only (not hostname). Root cause kept (`INSUFFICIENT_CONFIRMED_BALANCE`, `SIMULATION_REVERT`, `USER_REJECTED` warning, …). Message is `External {action} failed — hostname — reason`. Unknown calldata → `External dApp transaction failed`. |
| **DApp** | External `signMessage` / `personal_sign` / `eth_sign` / typed-data fail or reject | `EXTERNAL_SIGN_MESSAGE_UNSUPPORTED`, `EXTERNAL_SIGN_MESSAGE_REJECTED` (warning), `EXTERNAL_SIGN_MESSAGE_FAILED`. Off-chain — not a swap, no hash, no fee, nothing broadcast. |
| **DApp** | First-time inject connect and disconnect | `DAPP_CONNECT` / `DAPP_DISCONNECT` — **Connections only** |
| **RPC** | Gateway host failed and another host completed the request | Warning. `RPC_TIMEOUT`, `RPC_RATE_LIMITED`, `RPC_AUTH_REQUIRED`, `RPC_DNS_FAILURE`, `RPC_ENDPOINT_NOT_FOUND`, `RPC_SERVER_ERROR`, `RPC_MALFORMED_RESPONSE`, `RPC_CHAIN_MISMATCH`, `RPC_METHOD_UNSUPPORTED`, `RPC_WEBSOCKET_FAILURE`. Copy: `{chain} RPC host unavailable; fallback succeeded.` Log + Alerts, not Errors. |
| **RPC** | Every eligible host failed | Error `RPC_ALL_HOSTS_FAILED`. `{chain} RPC request failed — no configured host completed the request.` Log + Errors + Alerts. |
| **Bridge** | Internal bridge execute catch | Same presenter family on the send/fee legs |
| **Ledger** | Device connect / disconnect failures (including Solana Ledger) | `LEDGER_DISCONNECTED`, `LEDGER_WRONG_APP`, `LEDGER_BLIND_SIGN_REQUIRED`, `LEDGER_ERROR` |
| **Ledger** | Sequential broadcast **proof** — first host reject after a signed raw | `PENDING_BALANCE_RESERVED` (queued native), `INSUFFICIENT_CONFIRMED_BALANCE`, nonce / revert — the row is written **before** failover stops |

Send / swap / bridge / Ledger still write errors. **DApp connect/disconnect** appear **only** on **Connections**. External dApp **transaction / sign** failures appear on Log / Errors / Alerts (warnings skip Errors). RPC host failures never appear in Connections.

**External action proof (0.11.212):** hostname, page URL, and dApp brand do **not** prove a swap. Proven actions require an allowlisted function selector (ERC-20 `approve`/`transfer`, WETH `deposit`/`withdraw` on a known wrapped-native, Uniswap/Pancake/Velodrome swap or liquidity selectors) or a recognized Solana program **and** instruction (pump.fun buy/sell vs create; System/SPL transfer; Jupiter program). Unknown selector, unknown program, or undecodable instruction → generic `External dApp transaction failed`. Action codes (`EXTERNAL_SWAP_FAILED`, `EXTERNAL_APPROVAL_FAILED`, …) are stored when there is no better root cause; a useful root cause is never replaced.

**RPC host logging (0.11.212):** sequential failover is unchanged. A failed host + successful fallback is a **warning**, not a failed transaction. Deterministic account/nonce/revert rejects are **not** host failures. Host identity is a provider label (`Alchemy Polygon`, `polygon-rpc.com`) — never a URL, path, query, or API key. Repeated identical host failures are suppressed for 5 minutes (plus the 8s row dedupe). Logging is fire-and-forget and does not add RPC requests.

**Reserved subsystem names** (the logger accepts them; these paths do **not** write rows yet — not bugs, not wired):

| Subsystem | Intended later events |
|-----------|------------------------|
| **DApp** *(partial)* | Connect / disconnect / external send-sign / message-signing **are** wired. Unknown-chain switch and WalletConnect-local sign catches are not fully shared yet. |
| **Holdings** | Portfolio persist / ticker / discovery failures |
| **History** | History refresh / explorer failures |
| **Vault** | Lock / unlock / password wrap failures |
| **Settings** | RPC override save / identity check failures |
| **Wallet** | Generic catch-all |

Successful sends, swaps, and bridges **do not** write a Logs row. Hash-without-receipt is a **Submitted** lifecycle on the send/swap UI, not an error log. Quote / fee / token-lookup / clipboard-guard / result-tracker noise is also not logged.

### 9.3 What Logs never store

- Seeds, mnemonics, passwords, private keys, API keys
- Signed raw transactions or calldata
- Full addresses as a required field (messages are already presenter-redacted)
- Stack traces
- Complete RPC URLs, URL paths, query strings, API keys, project IDs
- RPC request parameters, signed raw transactions, message-signing payloads
- RugWatch mint / wallet databases (this console is wallet diagnostics only)

Optional safe extras (0.11.212–0.11.213): `action`, `operation`, `providerLabel`, `fallbackSucceeded`, `requestId`, `correlationId`, `recoveryOf`, `lifecycleState`.

**Capacity (0.11.213):** default **500** rows, optional Advanced **1,000**. FIFO drops the oldest. The Logs UI initially paints **150** newest rows and **Load earlier logs** adds 150 more. See **§9.3.1** for **Hide routine success** and **Keep more logs**. Recovery events are `info` and only write if a matching earlier failure exists (`recoveryOf`). This is structured coverage for verified paths — **not** a crash reporter or `console.log` interceptor.

### 9.3.1 Hide routine success (Settings)

**Location:** Settings → **Hide routine success**  
**Copy:** Keep errors and recoveries; hide submitted/confirmed info rows  
**Default:** on. From **0.11.233** the switch is restored from `chrome.storage.local` when the popup or full-page wallet opens (one local read; no extra RPC). Closing the popup does not reset it.

It is a **display filter**. It does not delete rows, does not stop writing them, and does not change the toast or the Send / Swap / Bridge status line. Turn it **off** and any stored matching rows come back. **Clear** still deletes by the selected tab.

**On (default):** hide a row only when it is **both** severity `info` **and** one of these six codes:

| Code | Meaning |
|------|---------|
| `TX_SUBMITTED` | Transaction submitted |
| `TX_CONFIRMED` | Transaction confirmed |
| `SWAP_APPROVAL_CONFIRMED` | Token approval confirmed before an in-wallet swap |
| `BRIDGE_DEST_COMPLETED` | Bridge arrived on the destination chain |
| `SETTINGS_RPC_OVERRIDE_SAVED` | RPC override saved |
| `STORAGE_MIGRATION_COMPLETED` | Storage migration finished |

That is the whole hide list. Not every green / info row.

**Off:** those six codes are shown if they were stored.

**Always kept (switch does not hide them):** every `critical`, `error`, and `warning`; recoveries (`RPC_HOST_RECOVERED`, `TX_CONFIRMATION_RECOVERED`, `HOLDINGS_BALANCE_RECOVERED`, other `*_RECOVERED`); other info that is not on the six-code list (lock / session, History reconcile, cancel / speed-up accepted, `DAPP_CONNECT` / `DAPP_DISCONNECT`).

**Which Logs tabs change**

| Tab | With the switch on |
|-----|--------------------|
| **Log** | Those six success rows disappear. This is the tab the Settings copy is talking about. |
| **Errors** | Unchanged — critical + error only |
| **Alerts** | Unchanged — critical + error + warning |
| **Warnings** | Unchanged — vault / dApp / seed / outgoing warnings |
| **Connections** | Unchanged — connect / disconnect are not on the hide list |

The top pills use the filtered list, so the **events** count can drop when success rows are hidden. Errors and warnings counts stay the same.

**What turning it off does not do**

- It does **not** start logging extra events.
- A successful **external DEX** swap (Uniswap, Pancake, and other sites) still writes **no** success row. `TX_CONFIRMED` / `SWAP_APPROVAL_CONFIRMED` are catalogued; they are not wired on the external-DEX path. Off still will not show “swap confirmed” for a good site swap.
- Failures, cancels, and review-vs-signed mismatches on that path still appear (error / warning / critical). The switch does not hide them.
- In-wallet Send / Swap / Bridge success is still primarily the on-screen status (and History). Most successful in-wallet trades do not write a Logs row. If `TX_SUBMITTED` was stored, off will show that submitted row — not a separate “swap confirmed” unless that code was actually written.

**Sibling switch — Keep more logs:** off = last **500** stored; on = last **1,000**. That is storage size (FIFO). It is not a display filter. From **0.11.233** it also persists across popup close.

### 9.4 Four-level severity (0.11.214)

Severity is a **presentation and filter** field. Canonical codes (`BROADCAST_UNCERTAIN`, `NONCE_TOO_LOW`, `RPC_RATE_LIMITED`, …) stay on the row. Color names never replace a useful error code.

Red is **rare**. Do not describe every failure as red. Red is reserved for uncertain fund movement, security/integrity failures, or similarly critical conditions.

| Stored | Label | Color | Meaning |
|--------|--------|--------|---------|
| `critical` | CRITICAL | red `--log-critical` `#c98a8e` | Possible fund/data risk, uncertain transaction state, or security-integrity failure |
| `error` | ERROR | orange `--log-error` `#c4a06a` | Operation definitely failed; wallet knows funds/state remain safe |
| `warning` | WARNING | yellow `--log-warning` `#c4b478` | Degraded service, delay, fallback, expected rejection, or user action required |
| `info` | INFO | green `--log-info` `#6eae90` | Normal operation, success, or recovery |

Color is never the only signal. Every row shows a `CRITICAL` / `ERROR` / `WARNING` / `INFO` text label. The four variables live on `#panel-logs` and do not change the rest of the wallet palette. Hex values are one notch quieter than the 0.11.214 set.

**Contrast:** Logs panels stay charcoal (`#121a24`) in both dark and light modes. Approximate contrast vs that background: critical ~6.2:1, error ~7.4:1, warning ~9.4:1, info ~8.3:1. All meet WCAG AA for normal text.

#### Central mapper

`severityForDiagnostic({ code, lifecycleState, signed, broadcastAttempted, broadcastAccepted, fallbackSucceeded, recovered, securityImpact })` in `diag-severity.js`. `sw-diag-log.js` calls it on sanitize / persist. UI renderers only consume the stored field.

Ambiguous-case rules (structured facts win over raw message text):

| Fact | Severity |
|------|----------|
| `recovered` / `recoveryOf` | info |
| `securityImpact` or `lifecycleState === signed_uncertain` | critical |
| Recipient / spender / value / nonce / payload mismatch; duplicate sign/broadcast; integrity failure | critical |
| Valid transaction hash | info (Submitted), never Failed |
| Delayed receipt | warning |
| Ambiguous signed broadcast without a reliable hash (`BROADCAST_UNCERTAIN`) | critical |
| Explicit pre-submission rejection (`BROADCAST_REJECTED`, insufficient, nonce, sim revert) | error |
| User rejection / cancel | warning — never orange or red |
| Successful recovery | info, and only once |
| One RPC host failed and fallback succeeded | warning |
| All hosts exhausted | error |
| All hosts exhausted after an already-signed broadcast with no accepted hash | critical |
| Unknown legacy `error` | stays error — never promoted to critical without proof |
| Already-stored `critical` on a known remapped code | stays critical (idempotent) |

#### Code-to-severity matrix

**Critical (red)** — every code in `diag-severity.js` `CRITICAL`. These appear on Log, Errors, Alerts, and (when they are Warnings-tab events) Warnings. None of them mean “seed definitely stolen” or “malware definitely present.”

| Code | Meaning |
|------|---------|
| `BROADCAST_UNCERTAIN` | A send/swap was signed or submitted; the RPC did not return a reliable result (no trusted hash). Check History / explorer before retrying. Do not re-sign blindly. |
| `TX_BROADCAST_UNCERTAIN` | Same class as above, Logs catalog name for send/swap lifecycle. Broadcast status is uncertain. Check History before retrying. |
| `ONCHAIN_REVERT` | The transaction executed on-chain and reverted. Funds typically return minus gas. |
| `TX_REVERTED` | Same class: transaction reverted on-chain (catalog name). |
| `REVIEW_RECIPIENT_MISMATCH` | The signed `to` / recipient did not match what you reviewed. Warns, then asks to proceed. |
| `REVIEW_SPENDER_MISMATCH` | The approval spender did not match what you reviewed. Warns, then asks to proceed. |
| `REVIEW_VALUE_MISMATCH` | The native-token value did not match what you reviewed. Warns, then asks to proceed. |
| `REVIEW_TOKEN_AMOUNT_MISMATCH` | The token amount did not match what you reviewed. Warns, then asks to proceed. |
| `SIGNED_NONCE_MISMATCH` | The nonce changed after you reviewed the transaction. Warns, then asks to proceed. |
| `SIGNED_PAYLOAD_MISMATCH` | The signed payload did not match the reviewed transaction. Warns, then asks to proceed. |
| `REVIEW_ACTION_MISMATCH` | The action (swap/send/approve/…) changed after review. Warns, then asks to proceed. |
| `CHAIN_ID_MISMATCH_SIGN` | The chain ID changed after you reviewed the transaction. Warns, then asks to proceed. |
| `HASH_PAYLOAD_CONFLICT` | A returned transaction hash does not match the signed payload. Warns, then asks to proceed. |
| `DUPLICATE_SIGNING` | A second signing request was not the same reviewed payload. Blocked. |
| `DUPLICATE_BROADCAST` | A second broadcast used a different payload. Blocked. |
| `PROVIDER_DISAGREEMENT_AFTER_SIGN` | RPC providers disagreed about the transaction after it was signed. Check History before retrying. |
| `ORIGIN_INTEGRITY_FAILURE` | The request origin could not be verified against the connected site. Blocked. |
| `SIGNING_RESERVATION_INTEGRITY_FAILURE` | Signing-reservation ownership changed unexpectedly. The request was blocked. |
| `UNAUTHORIZED_SIGNING_REQUEST` | A signing request was not authorized for this account. Blocked. |
| `UNEXPECTED_NATIVE_TRANSFER` | Local simulation showed a native-token transfer that was not the reviewed action. Classifier-ready. |
| `UNEXPECTED_TOKEN_TRANSFER` | Local simulation showed a token transfer that was not the reviewed action. Classifier-ready. |
| `UNEXPECTED_APPROVAL` | Local simulation showed an approval that was not the reviewed action. Classifier-ready. |
| `UNEXPECTED_MULTIPLE_RECIPIENTS` | Local simulation showed more than one recipient. Classifier-ready. |
| `UNEXPECTED_CONTRACT_CREATION` | Local simulation showed contract creation, which was not the reviewed action. Classifier-ready. |
| `SIMULATION_INTENT_MISMATCH` | Local simulation output conflicted with the action you reviewed. Classifier-ready. |
| `VAULT_STORAGE_CHANGED_UNEXPECTEDLY` | The encrypted vault changed without a Smart Wallet owner-write stamp or matching `swWrite`. Owner open/close/lock/unlock/settings do not create this. Does not lock Send. |
| `VAULT_AUTH_TAG_INVALID` | Vault authentication failed (AES-GCM). Smart Wallet could not open the encrypted vault. Unlock-form wrong password is not logged per attempt. |
| `VAULT_DECRYPTION_FAILED` | Vault decryption failed and was not classified as an auth-tag failure. |
| `VAULT_STRUCTURE_INVALID` | Decrypted vault bytes were not a valid structure. Secret material was not loaded. |
| `VAULT_INTEGRITY_FAILURE` | Generic vault integrity stop. Prefer a more specific vault code when one applies. |
| `VAULT_WRITE_AUTHORITY_MISMATCH` | A vault write did not match the authorized writer revision. |
| `VAULT_MIGRATION_FAILED` | Vault migration failed. The previous encrypted vault was left untouched. |
| `VAULT_STORAGE_FAILED` | An encrypted-vault persist/storage operation failed. Not a ciphertext-identity warning. |
| `SENSITIVE_PLAINTEXT_PERSISTENCE_DETECTED` | Seed/key plaintext was detected at a persistence boundary and was **not** stored. Classifier-ready. |
| `SENSITIVE_DATA_RENDER_CONTEXT_INVALID` | Sensitive data was blocked from rendering outside the authorized seed-reveal / extension screen. Classifier-ready. |
| `FEE_COLLECTED_BEFORE_CONFIRM` | Mapper-listed: a platform fee would have been taken before on-chain confirm. Verified fee paths must not do this. |
| `SECURITY_POLICY_FAILURE` | Mapper-listed generic security-policy failure. Prefer a more specific code when one applies. |

**Fact-based promotion (not a fixed code in the `CRITICAL` map):** `RPC_ALL_HOSTS_FAILED` is stored as **critical** only when `signed && broadcastAttempted && !broadcastAccepted` (already-signed broadcast, every host failed, no accepted hash). Otherwise it stays **error**.

**Error (orange)**

Send / nonce: `INSUFFICIENT_CONFIRMED_BALANCE`, `INSUFFICIENT_GAS`, `PENDING_BALANCE_RESERVED`, `NONCE_TOO_LOW`, `NONCE_TOO_HIGH`, `NONCE_PENDING_OCCUPIES`, `NONCE_GUARD_BLOCKED`, `REPLACEMENT_UNDERPRICED`, `REPLACEMENT_FIELD_INVALID`, `BROADCAST_REJECTED`, `TX_BROADCAST_REJECTED`, `SIMULATION_REVERT`, `SIMULATION_REVERTED`, `TOKEN_CONTRACT_REVERT`, `PROVIDER_REQUEST_TIMEOUT`, `TRANSACTION_ALREADY_KNOWN`

RPC: `RPC_AUTH_REQUIRED`, `RPC_UNAVAILABLE`, `RPC_DNS_FAILURE`, `RPC_ENDPOINT_NOT_FOUND`, `RPC_SERVER_ERROR`, `RPC_MALFORMED_RESPONSE`, `RPC_CHAIN_MISMATCH`, `RPC_METHOD_UNSUPPORTED`, `RPC_ALL_HOSTS_FAILED` (unless the signed-broadcast rule above)

Ledger / build: `LEDGER_DISCONNECTED`, `LEDGER_WRONG_APP`, `LEDGER_BLIND_SIGN_REQUIRED`, `LEDGER_ERROR`, `SIGNING_FAILED` (not user reject), `SERIALIZATION_FAILED`, `FEE_FIELD_INVALID`

Swap / bridge / external: `SWAP_APPROVAL_FAILED`, `SWAP_APPROVAL_UNCONFIRMED`, `SWAP_ALLOWANCE_READ_FAILED`, `SWAP_CANDIDATES_EXHAUSTED`, `ALL_ROUTES_SIM_FAILED`, `ROUTE_VALIDATION_FAILED`, `SWAP_SIMULATION_FAILED`, `TOKEN_TRANSFER_FAILED`, `TOKEN_TRANSFER_FROM_FAILED`, `SWAP_TOKEN_RESTRICTED`, `INSUFFICIENT_ALLOWANCE`, `ACCESS_CONTROL_FAILED`, `ARITHMETIC_PANIC`, `SWAP_FEE_ON_TRANSFER`, `BRIDGE_SOURCE_SEND_FAILED`, `BRIDGE_REVERTED`, `SWAP_FEE_SETTLEMENT_FAILED`, `BRIDGE_FEE_SETTLEMENT_FAILED`, `EXTERNAL_SWAP_FAILED`, `EXTERNAL_APPROVAL_FAILED`, `EXTERNAL_REVOKE_FAILED`, `EXTERNAL_WRAP_FAILED`, `EXTERNAL_UNWRAP_FAILED`, `EXTERNAL_LIQUIDITY_FAILED`, `EXTERNAL_TRANSFER_FAILED`, `EXTERNAL_DAPP_TRANSACTION_FAILED`, `EXTERNAL_SIGN_MESSAGE_FAILED`, `SWAP_MIN_OUTPUT`, `SWAP_DEADLINE_EXPIRED`, `SWAP_REVERTED`, `BRIDGE_ROUTE_UNAVAILABLE`, `BRIDGE_APPROVAL_FAILED`, `BRIDGE_PROVIDER_UNAVAILABLE`

Chain-specific / persist: `SOLANA_SIMULATION_FAILED`, `SOLANA_ATA_CREATION_FAILED`, `SOLANA_SPL_TRANSFER_FAILED`, `SOLANA_RENT_REQUIRED`, `EXTERNAL_TOKEN_ACCOUNT_CLOSE_FAILED`, `EXTERNAL_RENT_RECLAIM_FAILED`, `EXTERNAL_REWARD_CLAIM_FAILED`, `BTC_UTXO_SELECTION_FAILED`, `BTC_SERIALIZATION_FAILED`, `BTC_INSUFFICIENT_CONFIRMED`, `BTC_DUST_REJECTED`, `BTC_ADDRESS_NETWORK_MISMATCH`, `SUI_GAS_COIN_FAILED`, `SUI_DRY_RUN_FAILED`, `SUI_CONSTRUCTION_FAILED`, `SUI_INSUFFICIENT`, `SUI_COIN_OBJECT_FAILED`, `SUI_OBJECT_VERSION_CONFLICT`, `HISTORY_STORAGE_WRITE_FAILED`, `HISTORY_RPC_REFRESH_FAILED`, `HOLDINGS_PERSIST_FAILED`, `HOLDINGS_NATIVE_RPC_FAILED`, `HOLDINGS_TOKEN_RPC_FAILED`, `SETTINGS_PERSIST_FAILED`, `STORAGE_MIGRATION_FAILED`, `SETTINGS_RPC_IDENTITY_FAILED`, `SETTINGS_INVALID_ENDPOINT`, `SETTINGS_MISSING_CHAIN`

**Warning (yellow)**

`USER_REJECTED`, `USER_CANCELLED`, `SIGNATURE_REJECTED`, `NONCE_GUARD_WAIT`, `CONFIRMATION_DELAYED`, `CONFIRMATION_TIMEOUT`, `TX_CONFIRMATION_DELAYED`, `QUOTE_EXPIRED`, `QUOTE_CHANGED`, `STALE_INTENT`, `NO_ROUTE`, `NO_ROUTE_FOR_AMOUNT`, `NO_CONNECTOR_PATH`, `SWAP_NO_ROUTE`, `SWAP_QUOTE_EXPIRED`, `SWAP_QUOTE_CHANGED`, `RPC_RATE_LIMITED`, `RPC_TIMEOUT`, `RPC_COOLDOWN_ACTIVATED`, `RPC_WEBSOCKET_FAILURE`, `HOLDINGS_USD_UNAVAILABLE`, `HOLDINGS_METADATA_UNAVAILABLE`, `HOLDINGS_STALE_BALANCE_USED`, `HOLDINGS_STALE_PRICE_USED`, `HOLDINGS_PRICE_RATE_LIMITED`, `HOLDINGS_PRICE_MALFORMED`, `HOLDINGS_TOKEN_DECODE_FAILED`, `HOLDINGS_INVALID_DECIMALS`, `BRIDGE_DEST_DELAYED`, `BRIDGE_DEST_STATUS_DELAYED`, `BRIDGE_ROUTE_EXPIRED`, `BRIDGE_QUOTE_CHANGED`, `BRIDGE_PROVIDER_RATE_LIMITED`, `SWAP_FEE_PENDING`, `BRIDGE_FEE_PENDING`, `DAPP_UNSUPPORTED_METHOD`, `EXTERNAL_SIGN_MESSAGE_UNSUPPORTED`, `EXTERNAL_SIGN_MESSAGE_REJECTED`, `DAPP_UNKNOWN_CHAIN`, `DAPP_UNAUTHORIZED_ACCOUNT`, `DAPP_MALFORMED_REQUEST`, `DAPP_ORIGIN_BLOCKED`, `DAPP_SIGNING_CONFLICT`, `DAPP_APPROVAL_REJECTED`, `DAPP_TX_REJECTED`, `VAULT_SESSION_EXPIRED`, `VAULT_AUTOLOCK`, `VAULT_LOCKOUT`, `WALLET_SIGNING_RESERVATION_CONFLICT`, `SETTINGS_RPC_OVERRIDE_REJECTED`, `SETTINGS_TOKEN_IMPORT_FAILED`, `SETTINGS_VERSION_MISMATCH`, `SETTINGS_MODULE_PIN_MISMATCH`, `HISTORY_EXPLORER_REFRESH_FAILED`, `HISTORY_RECEIPT_LOOKUP_FAILED`, `HISTORY_MISSING_ONCHAIN`, `HISTORY_CACHE_STALE`, `TX_DROPPED`, `SWAP_PROVIDER_RATE_LIMITED`, `SWAP_PROVIDER_COOLDOWN`, `SWAP_FALLBACK_SELECTED`, `PROVIDER_RATE_LIMITED`, `SOLANA_BLOCKHASH_EXPIRED`, `SOLANA_SIGNATURE_REJECTED`, `SOLANA_VERSIONED_TX_UNSUPPORTED`, `BTC_FEE_ESTIMATE_UNAVAILABLE`

**Info (green)**

`TX_SUBMITTED`, `TX_CONFIRMED`, `TX_CANCEL_ACCEPTED`, `TX_CANCEL_CONFIRMED`, `TX_SPEEDUP_ACCEPTED`, `TX_SPEEDUP_CONFIRMED`, `TX_NONCE_QUEUE_CLEARED`, `TX_NONCE_RECONCILED`, `TX_REPLACED`, `RPC_HOST_RECOVERED`, `RPC_WEBSOCKET_RECOVERED`, `HISTORY_RECONCILED`, `HISTORY_DUPLICATE_RECONCILED`, `HISTORY_REPLACEMENT_GROUP_UPDATED`, `HISTORY_CANCEL_LABEL_REPAIRED`, `HISTORY_RECOVERY_COMPLETED`, `HOLDINGS_BALANCE_RECOVERED`, `HOLDINGS_PRICE_RECOVERED`, `BRIDGE_DEST_COMPLETED`, `SWAP_APPROVAL_CONFIRMED`, `SWAP_CONFIRMATION_RECOVERED`, `TX_CONFIRMATION_RECOVERED`, `VAULT_LOCKED`, `SETTINGS_RPC_OVERRIDE_SAVED`, `STORAGE_MIGRATION_COMPLETED`, `DAPP_CONNECT`, `DAPP_DISCONNECT`, `WALLET_SW_STATE_RESTORED`, `WALLET_SIGNING_RESERVATION_RECOVERED`

#### Tabs, Clear, migration, privacy

| Tab | Shows | Hides |
|-----|--------|--------|
| Log | critical, error, warning, info except Connections | `DAPP_CONNECT` / `DAPP_DISCONNECT` |
| Errors | critical + error | warning, info, Connections |
| Alerts | critical + error + warning | info, Connections |
| Warnings | dApp / vault / seed / plaintext-sink / unrecognized-outgoing codes | unrelated send/RPC and Connections |
| Connections | `DAPP_CONNECT` / `DAPP_DISCONNECT` only (connect green, disconnect red) | everything else |

One stored row may appear in several tabs. It is persisted once.

**Clear (selected tab, not the entire store):** Log removes non-Connections rows. Errors removes critical + error. Alerts removes critical + error + warning. Warnings removes Warnings-filter rows. Connections removes connect/disconnect. Other tabs keep their remaining rows. Shared rows disappear from every view they were in. Every clear immediately repaints all five views. Cap remains 500 default / 1,000 optional; FIFO drops the oldest.

**Legacy migration:** existing rows may only have `error` / `warning` / `info`. Known codes are remapped through `severityForDiagnostic` on read and write. Existing `error` + a critical code (`BROADCAST_UNCERTAIN`, …) becomes `critical`. Ordinary `error` stays `error`. `warning` / `info` stay unless structured facts prove otherwise. Unknown codes are **never** promoted to critical. Migration is idempotent: timestamps, ids, and repeat counts are kept; rows are not duplicated; `originalErr` is never written.

**Privacy (unchanged):** service-worker serialized writes, fire-and-forget, 8s dedupe, 5-minute host-failure suppress, newest-first, no telemetry, no extra RPC, no paint while Logs is closed. Never persist seeds, keys, passwords, API keys, credentialed RPC URLs, raw calldata, signed raw, signatures, message-signing payloads, full `Error`/`originalErr`, stack traces, or clipboard contents.

**Not changed in 0.11.214:** transaction construction, signing, nonce selection, sequential RPC failover, swap 45 bps, bridge 85 bps.

Shipped **0.11.145** (store + redaction). Redaction hardened **0.11.153**. In-wallet console **0.11.191–0.11.197**. Connections **0.11.204**. Newest-first **0.11.205**. Tab-scoped Clear **0.11.208**. SW-serialized writes + `originalErr` isolation **0.11.209**. External DEX swap failures on Logs **0.11.210**. External DEX **reason** on the Swap row **0.11.211**. External dApp intent + RPC host rows **0.11.212**. Expanded diagnostic catalog, 500/1000 cap, recovery, load-earlier **0.11.213**. Four-level severity **0.11.214**.

### 9.5 Warnings tab (0.11.216)

**Warnings reports observable request, origin, approval and transaction-integrity conditions. It does not provide general malicious-dApp reputation or guarantee that a contract is safe.**

It is another filter on the same store. A critical recipient mismatch appears in Log, Errors, Alerts, and Warnings, stored once. Severity colors are not remapped because a row is on Warnings.

Module: `dapp-security-warn.js` (`classifyDappSecurity`, `isWarningsTabEvent`). Severity still comes from `severityForDiagnostic`.

**Wording:** report the observable fact. Example: `Smart Wallet blocked this request because its signed payload did not match the transaction you reviewed.` Never: `Smart Wallet detected a malicious dApp.`

**What is wired today (local evidence only):**

| Signal | Code | Severity | Blocking? |
|--------|------|----------|-----------|
| Inject host not on allowlist (connect/sign) | `ORIGIN_NOT_ALLOWED` | warning | Record only |
| Frame origin ≠ top origin (`ancestorOrigins`) | `ORIGIN_FRAME_MISMATCH` | warning | Record only |
| Missing origin | `ORIGIN_METADATA_INVALID` | warning | Record only |
| Punycode `xn--` hostname | `ORIGIN_UNICODE_WARNING` | warning | Record only — not called malicious |
| `tx.from` ≠ active account | `UNAUTHORIZED_ACCOUNT_REQUEST` | error | Yes (existing 4001) |
| Unsupported typed-data chain | `UNAUTHORIZED_CHAIN_REQUEST` | error | Yes (existing 4902) |
| Unlimited ERC-20 approve | `UNLIMITED_APPROVAL_REQUESTED` | warning | No — user still reviews |
| Approve spender not on local router list | `UNKNOWN_APPROVAL_SPENDER` | warning | No |
| Reviewed vs post-approve snapshot differs | `REVIEW_*` / `SIGNED_*` / `CHAIN_ID_MISMATCH_SIGN` | critical | Warns, then asks to proceed (0.11.232) |
| Signing reservation already held | `DAPP_SIGNING_CONFLICT` | warning | Yes (existing) |
| External `signMessage` unsupported / rejected | `EXTERNAL_SIGN_MESSAGE_*` | warning | Existing |
| Site permission revoked normally | `DAPP_PERMISSION_REVOKED` | info | n/a |

**Not claimed live** unless the classifier is given those facts: unexpected simulation value-flow, general contract reputation, Solana instruction mismatch on every program, remote URL scanning.

**Clear:** Warnings removes only Warnings-filter rows. Shared rows leave Log/Errors/Alerts too. Connections stay. All five views repaint. Cap remains 500/1000.

**Home:** the Total balance line shows a red-dot count of stored **critical** Warnings-tab rows (`1`, `2`, …) plus **review logs**. Click opens Logs → Warnings. The badge hides when that count is 0 (including after Clear Warnings).

**Privacy:** same allowlist as Logs. No message bytes, signatures, calldata, or raw Errors.

**Mocked vs live:** `tools/test-dapp-security-warnings.js` is mocked. Live coverage is the wired rows above. No extra RPC or simulation is added to generate Logs.

### 9.6 Vault, seed-reveal, and unrecognized outgoing (0.11.217)

Warnings also shows vault, seed-reveal, plaintext-sink, and unrecognized outgoing events. Severity colors stay. Codes still go through `severityForDiagnostic`.

**What Smart Wallet can and cannot prove**

- Viewing a software-wallet seed requires temporary plaintext in the trusted extension context.
- A seed reveal is not itself a vault-integrity failure.
- Logs never store the seed, word count, passwords, clipboard contents, or decrypted vault bytes.
- AES-GCM authentication failure proves decryption/authentication failed — not who caused it, and not that the seed is compromised.
- Unrecognized outgoing activity does not prove key compromise. It may be another device, another wallet using the same seed, a previously approved dApp, or a local History gap.
- Ledger seed material never enters Smart Wallet.

**Seed-reveal metadata (wired):** `SEED_REVEAL_STARTED` (info), `SEED_REVEAL_CLOSED` / `SEED_REVEAL_AUTO_HIDDEN` (info), `SEED_CLIPBOARD_COPY` (warning, no content), `SEED_CLIPBOARD_CLEARED` (info), clear-fail (warning), reauth fail (warning, no password), Ledger context blocked (error).

**Vault (wired, 0.11.229):** `extractCanonicalEncryptedVault` compares only encrypted-vault identity (`v`, `kdf`, `alg`, `keyLen`, `iterations`, salt, iv, data, optional tag). AES-GCM’s authentication tag lives inside `data`. Container fields (lock, session, selected account/network, Ledger/holdings/token metadata, settings, RPC, last screen, cache, UI) are ignored.

The watcher loads the stored vault, adopts its identity once as the trusted baseline (including legacy vaults with no `swWrite`), then marks ready. It does not re-encrypt old vaults just to add a marker. `swWrite` is a one-time authorization (`id`, prior identity, next identity, operation, writer, expiry) stored in `smart_wallet_vault_auth_v1`. Every Smart Wallet persist of `smart_wallet_v1` (popup, full-page wallet, service worker, restore) also writes a short-lived **owner-write** stamp (`smart_wallet_vault_owner_write_v1`) for the vault identity being saved. The watcher treats a matching unexpired owner stamp as the owner, not an outside change. A valid `swWrite` still authorizes only that exact change. Popup and service worker re-read both stamps before deciding.

Same-ciphertext container rewrites (open/close/reopen, lock/unlock, auto-lock, network/account switch, Ledger link/select, settings/RPC/token/holdings/last-screen, reload, SW restart, cache, dApp permission metadata) create no vault warning and do not increment a repeat count. Authorized create/import/remove/password/re-encrypt/restore/migrate/update-secrets writes match old/new identity and produce no critical row.

An unauthorized ciphertext change records **one** `VAULT_STORAGE_CHANGED_UNEXPECTEDLY` (critical). It does not lock software wallets or block Send, Buy, Sell, Swap, or Bridge. The row does not claim the seed was stolen, malware is present, or funds moved. Ledger secrets are not implicated. The previous and new vault blobs are not overwritten automatically. Auth-tag / decrypt / structure failures stay `VAULT_AUTH_TAG_INVALID`, `VAULT_DECRYPTION_FAILED`, `VAULT_STRUCTURE_INVALID`. Historical critical rows are not auto-deleted; they may be relabeled `VAULT_CONTAINER_REWRITE_IGNORED` (info) only when a later canonical compare proves the ciphertext was identical. Stripping plaintext before persist → `SENSITIVE_PLAINTEXT_PERSISTENCE_BLOCKED` (error).

**Vault critical codes** (Warnings tab + Errors/Alerts because they are red; severity from `severityForDiagnostic`). None of these rows store ciphertext, salt, iv, tag, password, or seed. Unlock-form wrong password is **not** a logged row.

| Code | Severity | Wired today | What the row means |
|------|----------|-------------|--------------------|
| `VAULT_STORAGE_CHANGED_UNEXPECTEDLY` | critical | Yes — vault watcher | Canonical encrypted vault changed **without** a Smart Wallet owner-write stamp or matching `swWrite`. Owner actions (open/close, lock/unlock, settings, authorized re-encrypt) do not create this row. Does not lock Send/Buy/Sell. Does not prove seed theft, malware, or that funds moved. |
| `VAULT_AUTH_TAG_INVALID` | critical | Yes — session restore AES-GCM fail | Vault authentication failed. Smart Wallet could not open the encrypted vault. Unlock-form wrong password is not logged per attempt. |
| `VAULT_DECRYPTION_FAILED` | critical | Classifier-ready | Decryption failed and was not classified as an auth-tag failure. |
| `VAULT_STRUCTURE_INVALID` | critical | Yes — decrypt JSON / secrets map | Decrypted bytes were not a valid vault structure. Secret material was not loaded. |
| `VAULT_INTEGRITY_FAILURE` | critical | Classifier-ready | Generic vault integrity stop. Prefer a more specific code when one applies. |
| `VAULT_WRITE_AUTHORITY_MISMATCH` | critical | Classifier-ready | A vault write did not match the authorized writer revision. Distinct from an unexplained ciphertext change. |
| `VAULT_MIGRATION_FAILED` | critical | Classifier-ready | Vault migration failed. The previous encrypted vault was left untouched. |
| `VAULT_STORAGE_FAILED` | critical | Catalogued | Encrypted-vault persist failed. Not a ciphertext-identity warning. |

**Related vault rows that are not critical**

| Code | Severity | What the row means |
|------|----------|--------------------|
| `SENSITIVE_PLAINTEXT_PERSISTENCE_BLOCKED` | error | Software secrets could not be sealed; they were stripped before persist. |
| `VAULT_LOCKOUT` | warning | Lockout after repeated failed unlock attempts. |
| `VAULT_RECOVERED` | info | Vault access recovered after a previous integrity failure. |
| `VAULT_CONTAINER_REWRITE_IGNORED` | info | Used only when a later canonical compare **proves** the ciphertext was identical. Not emitted for ordinary same-vault persists. Historical false critical rows are not auto-relabeled. |

**Unrecognized outgoing (wired on existing History refresh):** first load of a chain/account is a seed pass (no warnings). A later new outgoing hash from the selected account that is not in local txs / accepted hashes is yellow: `UNRECOGNIZED_OUTGOING_ACTIVITY`. Incoming transfers and incoming bridges do not trigger it. No extra RPC is added.

**Privacy:** same Logs allowlist. Never persist seeds, keys, passwords, vault ciphertext/tags, clipboard, calldata, signed raw, or raw Errors.

### 9.6.1 Seed / private-key plaintext boundaries

Smart Wallet detects seed or private-key plaintext reaching a **prohibited** location at its guarded boundaries. It cannot detect every possible exposure.

**It may warn through these codes** (Warnings tab; severity from `severityForDiagnostic`):

| Code | Severity | What the row means |
|------|----------|--------------------|
| `SENSITIVE_PLAINTEXT_PERSISTENCE_DETECTED` | critical | Sensitive plaintext was detected at a persistence boundary and was **not** stored. |
| `SENSITIVE_DATA_RENDER_CONTEXT_INVALID` | critical | Sensitive data was **blocked** from rendering outside the authorized seed-reveal / extension screen. |
| `SENSITIVE_PLAINTEXT_PERSISTENCE_BLOCKED` | error | Smart Wallet blocked plaintext **before** it was saved. **Wired today** in `buildDiskState`: if software secrets cannot be sealed, they are stripped and this row is recorded. |
| `SENSITIVE_DATA_LOGGING_BLOCKED` | error | Smart Wallet prevented sensitive data from entering Logs. The persist allowlist always redacts seed/key/password fields. This named row is recorded only when the sink classifier is invoked with that fact. |

`SENSITIVE_PLAINTEXT_PERSISTENCE_DETECTED` and `SENSITIVE_DATA_RENDER_CONTEXT_INVALID` are classifier-ready (they appear on Warnings if recorded). They are **not** claimed as firing on every persist or every paint.

It should **not** warn merely because you intentionally selected **View Seed Phrase**. Viewing a software-wallet seed requires temporary plaintext inside the trusted extension screen. That path records `SEED_REVEAL_STARTED` (info), then close / auto-hide metadata — not a persistence or render-boundary failure.

**Ledger accounts.** Ledger seed material never enters Smart Wallet. If the active account is Ledger and you see a seed-phrase warning, you can **most likely ignore it**:

- Tapping View Seed Phrase on Ledger records `SEED_REVEAL_CONTEXT_BLOCKED` (error) and shows that the phrase is not stored here. That is the expected block, not a leak.
- From **0.11.228 / 0.11.229**, clicking away and reopening, or a normal STATE persist of the same ciphertext, does **not** create `VAULT_STORAGE_CHANGED_UNEXPECTEDLY`. That code is reserved for a real encrypted-vault change without a verified authorized write. It is still not proof that a Ledger phrase is in the extension. Pre-0.11.228 builds could raise that row as a watcher false positive on unstamped vaults or container rewrites. Existing rows from those builds are not auto-deleted.

If this same wallet also holds a **software-wallet** seed, treat `SENSITIVE_PLAINTEXT_PERSISTENCE_DETECTED` or `SENSITIVE_DATA_RENDER_CONTEXT_INVALID` as applying to that software seed — not to the Ledger device.

**It cannot reliably detect:**

- Malware reading process memory
- Screen capture or someone photographing the phrase
- Another browser extension reading the screen
- Clipboard history outside Smart Wallet’s control
- A seed entered into another wallet or website
- Plaintext exposure through a code path that has no guard
- Secrets stolen before the detector was installed

**Accurate claim:** Smart Wallet can detect or block plaintext at its guarded storage, logging, messaging, and rendering boundaries. It cannot guarantee that the seed has never been exposed.

If it reports **confirmed** plaintext persistence (`SENSITIVE_PLAINTEXT_PERSISTENCE_DETECTED`) or rendering in an untrusted context (`SENSITIVE_DATA_RENDER_CONTEXT_INVALID`), treat that as **critical** for any software-wallet seed in this extension: stop using that software-wallet seed and move funds to a newly generated wallet through a trusted environment. A successful **block** (`SENSITIVE_PLAINTEXT_PERSISTENCE_BLOCKED` / `SENSITIVE_DATA_LOGGING_BLOCKED`) means the guarded path refused the write — not that the seed is known stolen.

### 9.7 Warnings that may be triggered by owner

Some **critical** and **warning** codes are written for an outsider, a deceptive request, or a storage/wallet fault. The owner can still produce those same rows by accident. A row on this list does **not** prove theft, malware, or that someone else has the seed.

**Warning:** A Logs row that the owner can trigger is **not** full proof of vault integrity, key compromise, or an outsider. Check the tables below and confirm you did **not** trigger it by accident (reload, double-click, account/network switch, another device with the same seed, profile restore, crash mid-save) before treating it as an integrity failure.

This section is only those outsider / fault-shaped codes that also have a realistic **owner accident** path. Ordinary owner-only rows (Reject, auto-lock, quote expired, RPC rate-limit, on-chain revert after you signed, unlimited Uniswap approve, allowlist miss) are not listed here.

**Not an owner accident** if they appear: `SENSITIVE_PLAINTEXT_PERSISTENCE_DETECTED`, `SENSITIVE_DATA_RENDER_CONTEXT_INVALID`, `FEE_COLLECTED_BEFORE_CONFIRM`, `SECURITY_POLICY_FAILURE`. Treat those as a wallet-path bug or a real boundary hit.

#### Easy to trip yourself

| Code | Typical owner accident |
|------|------------------------|
| `REVIEW_RECIPIENT_MISMATCH` | Reload the popup, switch account, or the site re-quotes while Confirm is open |
| `REVIEW_SPENDER_MISMATCH` | Same, during an approve |
| `REVIEW_VALUE_MISMATCH` | Gas/fee refresh changed the native amount after you reviewed |
| `REVIEW_TOKEN_AMOUNT_MISMATCH` | Slippage/quote refresh changed the token amount |
| `REVIEW_ACTION_MISMATCH` | Site/wallet re-labeled the same click (swap vs approve) mid-prompt |
| `SIGNED_PAYLOAD_MISMATCH` | Popup closed/reopened; fees rebuilt; you signed a newer assembled tx |
| `SIGNED_NONCE_MISMATCH` | Another send landed or the network nonce moved while you waited |
| `CHAIN_ID_MISMATCH_SIGN` | You changed network in the wallet bar while the dApp prompt was open |
| `DUPLICATE_SIGNING` | Double-click Confirm |
| `DUPLICATE_BROADCAST` | You hit send/retry after a timeout when the first broadcast already went |
| `DUPLICATE_SIGN_MESSAGE_REQUEST` | Double-click a sign-message prompt |
| `UNAUTHORIZED_SIGNING_REQUEST` | dApp still talking to account A after you switched to account B |
| `DAPP_UNAUTHORIZED_ACCOUNT` | Same account-switch |
| `SIGNING_RESERVATION_INTEGRITY_FAILURE` | Two wallet windows, or close the popup mid-Ledger/sign |
| `ORIGIN_INTEGRITY_FAILURE` | Uniswap/SPA remount; Chrome dropped the original tab channel |
| `ORIGIN_FRAME_MISMATCH` | Legit site using an iframe (embedded swap / WalletConnect) |
| `ORIGIN_CHANGED_AFTER_CONNECT` | Site hops `www` → `app`, or http → https |
| `PERMISSION_ORIGIN_MISMATCH` | Same origin hop after you already connected |
| `REJECTED_REQUEST_RESUBMITTED` | You rejected, then clicked the site button again |

#### Easy if you use more than this popup

| Code | Typical owner accident |
|------|------------------------|
| `UNRECOGNIZED_OUTGOING_ACTIVITY` | Same seed in another wallet or device, or History missed a local send |
| `ACCEPTED_HASH_NOT_LOCALLY_INITIATED` | Same |
| `OUTGOING_ACTIVITY_SOURCE_UNKNOWN` | History gap after reload / chain switch |
| `VAULT_STORAGE_CHANGED_UNEXPECTEDLY` | Profile restore, browser-profile sync, or two wallet copies writing. Pre-0.11.228 could also false-fire on click-away. |
| `VAULT_WRITE_AUTHORITY_MISMATCH` | Two Smart Wallet windows writing at once |

#### Possible on a messy but legitimate dApp

| Code | Typical owner accident |
|------|------------------------|
| `HASH_PAYLOAD_CONFLICT` | Speed-up / replace / RPC returned a different hash view |
| `UNEXPECTED_NATIVE_TRANSFER` | Aggregator route wraps/unwraps ETH as part of a real swap |
| `UNEXPECTED_TOKEN_TRANSFER` | Multi-hop / fee-on-transfer / extra hop you did not notice |
| `UNEXPECTED_APPROVAL` | Router does an approve inside the same flow |
| `UNEXPECTED_MULTIPLE_RECIPIENTS` | Fee recipient + you, or a split route |
| `UNEXPECTED_CONTRACT_CREATION` | Some routers deploy a temp contract |
| `SIMULATION_INTENT_MISMATCH` | Sim and UI disagree on a weird but real route |
| `DISPLAYED_ACTION_MISMATCH` | Aggregator selector not in the small local map |
| `DEX_ACTION_MISMATCH` | Shown as swap; selector is a custom/aggregator function |
| `EVM_SELECTOR_MISMATCH` | Same |
| `SOLANA_INSTRUCTION_MISMATCH` | Jupiter/other instruction set does not match the simple label |
| `APPROVAL_PAYLOAD_CHANGED` | Allowance already moved; site rebuilt the approve |
| `SIGN_MESSAGE_ORIGIN_MISMATCH` | Sign-in message from a subdomain |
| `SIGN_MESSAGE_DOMAIN_MISMATCH` | Message lists `uniswap.org` while the page is `app.uniswap.org` |
| `SIGN_MESSAGE_BINARY_WARNING` | Typed-data / hex that is not plain English |
| `SIGN_MESSAGE_CONTROL_CHARACTERS` | Odd UTF-8 in a real message |
| `ORIGIN_UNICODE_WARNING` | Rare legitimate IDN hostname |
| `DAPP_ORIGIN_BLOCKED` | You opened a real site that is blocked by policy |

#### Storage / crash — you interrupted the wallet

| Code | Typical owner accident |
|------|------------------------|
| `VAULT_STORAGE_FAILED` | Kill the browser mid-save, disk full, profile locked |
| `VAULT_MIGRATION_FAILED` | You started a vault/password migration and it died mid-way |
| `VAULT_AUTH_TAG_INVALID` | Crash mid-write or a bad backup restore |
| `VAULT_DECRYPTION_FAILED` | Same class (wrong-password clicks are **not** logged) |
| `VAULT_STRUCTURE_INVALID` | Corrupt or half-written vault after a crash/restore |
| `VAULT_INTEGRITY_FAILURE` | Same, generic stop |

---

## 10. Confirm truth (shared with Send and Swap)

`evmConfirmStateFromOutcome` (**0.11.152**):

| Receipt | Confirm state | UI | Platform fee |
|---------|---------------|----|--------------|
| `status = 1` | `CONFIRMED` | Confirmed | May collect (45 / 85 bps) |
| `status = 0` | `FAILED` | Failed on-chain | **Not due** |
| missing / not found | `CONFIRMING` | Submitted — confirming… | **Not due** until later receipt |

The injected provider **must not invent `0x1`**. Unknown chain throws. `PROVIDER_HINT_GEN` invalidates stale `eth_accounts` after account / chain / lock / disconnect / revoke.

---

## 11. Tests (docs names only)

| Script | Covers |
|--------|--------|
| `tools/test-tx-error-present.js` | Presenter codes + lifecycle sentences |
| `tools/test-extract-tx-error-present.js` | 0.11.154 extraction goldens |
| `tools/test-polygon-rpc-errors.js` | HTTP/DNS ≠ insufficient POL |
| `tools/test-ledger-broadcast-pending.js` | Stop on queued-balance reject |
| `tools/test-send-presign-pending.js` | estimateGas presenter before Nano |
| `tools/test-evm-revert-decoder.js` | Allowlisted selectors |
| `tools/test-swap-outcome.js` | Swap state / codes |
| `tools/test-diag-logs.js` | Cap, dedupe, redaction, non-blocking |
| `tools/test-diag-coverage.js` | Catalog coverage + recovery + cap (mocked) |
| `tools/test-diag-severity.js` | Four-level mapper, tabs, Clear, migration, FIFO (mocked) |
| `tools/test-dapp-security-warnings.js` | Warnings tab filter, classifier, Clear, no-malicious wording (mocked) |
| `tools/test-vault-security-warnings.js` | Vault / seed-reveal / unrecognized-outgoing Warnings (mocked) |
| `tools/test-tx-lifecycle.js` | Hash identity + CONFIRMING ≠ FAILED |

`tools/verify-extension.ps1` is the operator gate after any Error System edit.

**Mocked versus live:** `test-diag-severity.js`, `test-diag-logs.js`, `test-diag-coverage.js`, and `test-rpc-host-logs.js` are fully mocked (in-memory store, no network). Send / Swap / Bridge / Ledger / RPC / nonce / History suites remain mocked except where a script already labels a live estimate. 0.11.214 does not add RPC calls or live telemetry.

---

## 12. History (how the system was built)

| Version | What landed |
|---------|-------------|
| **0.11.137** | First `evm-error-classify.js`. Dead/auth Polygon hosts were being read as “not enough POL.” |
| **0.11.138** | Insufficient-funds only from numeric proof or the exact Geth phrase. Friendly string “Broadcast failed — not enough POL” is **not** proof. |
| **0.11.139** | Ledger cache must decode signed raw; RPC insufficient-funds ignored when leftover native is proven. |
| **0.11.140** | Presentation layer. `PENDING_BALANCE_RESERVED` vs `INSUFFICIENT_CONFIRMED_BALANCE`. |
| **0.11.141** | Deep `inspectError` walk. Prefer numeric fields over regex. |
| **0.11.142** | First deterministic queued-balance reject **stops** Ledger host failover. |
| **0.11.143** | Pre-sign estimateGas uses the presenter. No raw RPC in the send form. |
| **0.11.144** | Nested `info.error.message` is the classification source. Stamp pass-through. |
| **0.11.145** | Privacy-safe Logs. |
| **0.11.154** | Extract remaining send presentation into `tx-error-present.js`. Behavior unchanged. |
| **0.11.155** | Later-receipt swap fee uses confirm truth (Submitted ≠ fee due). Presenter unchanged. |
| **0.11.213** | Diagnostic catalog, 500/1000 cap, recovery, load-earlier. |
| **0.11.214** | Four-level Logs severity (`critical`/`error`/`warning`/`info`). Central mapper. Connections disconnect is green. |
| **0.11.215** | Same mapper. Log colors muted one notch (`#e07f85` / `#d4a05c` / `#d4be6a` / `#72c49a`). |
| **0.11.218** | Log colors muted again (`#c98a8e` / `#c4a06a` / `#c4b478` / `#6eae90`). Connections disconnect is red (`#c87a7e`). |
| **0.11.216** | Logs **Warnings** tab: observable dApp origin / approval / review-integrity events. Not reputation. |
| **0.11.217** | Warnings also covers vault integrity, seed-reveal metadata, plaintext-sink blocks, and unrecognized outgoing History. |
| **0.11.219** | Home Total balance red-dot count of stored critical Warnings-tab rows. |
| **0.11.220** | Home badge copy is **review logs** (was check logs). Docs: Clear deletes the **selected tab’s logs** (not the entire store). Seed plaintext-boundary claim + Ledger seed-warning guidance. |
| **0.11.221** | Logs chrome polish only: per-tab accents, outline Clear, gold Load Earlier, button/tab states. No filter, store, or severity change. |
| **0.11.222** | Light-mode Logs leftover fill matches dashboard `#121a24`. Visible Logs banner removed so the list has more space. |
| **0.11.223** | Logs leftover is `#3d3e46`. Home / Send / History / Accounts and the dock keep original light colors. Clear default matches hover. |
| **0.11.224** | Logs page network / address / account chips and dock hover use opaque Home plates so they do not go dark over `#3d3e46`. |
| **0.11.225** | Restored Logs selected-tab gold `#c9a227` border and list-box `#3d4d60` outline. Leftover fill stays `#3d3e46`. |
| **0.11.226** | Logs title through Clear logs wrapped in a rounded `#121a24` shell (`12px` radius, `#3d4d60` border). |
| **0.11.227** | Leftover is `#121a24`. Inner log well stays `#121a24`. `#3d3e46` is the outline around that well. |
| **0.11.228** | Vault watcher compares ciphertext identity, not the whole `smart_wallet_v1` container. Popup reopen / session saves no longer raise `VAULT_STORAGE_CHANGED_UNEXPECTEDLY`. Real unauthorized ciphertext changes still do. |
| **0.11.229** | Canonical extractor + durable one-time `swWrite` (old/new identity, op, writer, expiry). Historical false rows are not auto-deleted. Docs include a vault critical-code table. |
| **0.11.230** | Unauthorized vault warning no longer locks software wallets or blocks Send, Buy, Sell, Swap, or Bridge. Watcher still records one critical row. |
| **0.11.231** | Owner-write stamp on every Smart Wallet persist. `VAULT_STORAGE_CHANGED_UNEXPECTEDLY` only when the encrypted vault changes without that owner stamp (outside the wallet). Docs include the full critical-code table (every `CRITICAL` mapper code + meaning). |
| **0.11.232** | Critical review/sign mismatches no longer hard-block. They record a critical Logs row and ask **Proceed** or **Cancel**. |
| **0.11.233** | **Hide routine success** and **Keep more logs** persist across popup close / reopen. One local storage read on open. No extra RPC. Docs §9.3.1 explain the hide-success switch (what it hides, which tabs change, and that it does not log external-DEX “swap confirmed”). |
| **Docs 2026-08-16** | §9.7 **Warnings that may be triggered by owner** — outsider/fault-shaped codes that the owner can still produce by accident. A row the owner can trigger is not full proof of vault integrity; confirm it was not accidental. |

---

## 13. What this system does **not** do

- It does **not** retry, re-sign, or rebroadcast.
- It does **not** parallel-fan-out RPCs to “be sure.”
- It does **not** treat explorer lag as failure.
- It does **not** collect a platform fee on a hash-only / reverted swap.
- It does **not** phone home. Logs stay on the device.
- It does **not** replace Chain Registry / RPC Gateway / Transaction Manager.

---

## Related docs

| File | Role |
|------|------|
| [INTERNAL-DEX.md](./INTERNAL-DEX.md) | Internal swap architecture (quotes, routers, fee due) |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Whole-wallet build shape |
| [BUGS-AND-FIXES.md](./BUGS-AND-FIXES.md) | Bugs vs by-design vs fixed |
| [LOADS.md](./LOADS.md) | Ping / HTTPS / RPC counts |
| [MODULES.md](./MODULES.md) | Runtime module inventory |

---

*Not financial advice. Cryptocurrency involves risk of loss.*
