# Key protection in Smart Wallet

**Product:** Smart Wallet (Chrome / Opera MV3 browser extension)  
**Repo:** [whitebrick86-jpg/Smart-Wallet](https://github.com/whitebrick86-jpg/Smart-Wallet) (docs only)  
**Code snapshot:** **0.11.473** (Load-unpacked extension; source not published here)  
**Primary storage:** `smart_wallet_v1` — public account metadata + **encrypted vault**  
**Last updated:** 2026-08-23

This document describes **how software keys and passwords are protected** in the current product. It covers the **JavaScript vault** (legacy / unmigrated) and the **Rust/WASM vault** (after a verified migrate). RPC, price, quote and portfolio modules are **not intentionally given private keys**. The legacy JavaScript signing path may temporarily place signing secrets in the service-worker signer cache.

**Related:** [ARCHITECTURE.md](./ARCHITECTURE.md) (RPC, managers, tx lifecycle).

---

## Core statement (Rust-active)

While a Rust-active wallet is unlocked, JavaScript retains a WASM session handle rather than the password or private keys. The WASM session retains encrypted/session material sufficient to authorize operations. During routine signing, the required key material is temporarily decrypted or derived inside WASM, used to sign, and application-controlled buffers are zeroized afterward. Only the signature or signed transaction returns to JavaScript. Explicit secret reveal is an exception: the selected secret temporarily crosses into JavaScript and the DOM for display.

Application-controlled Rust password, blob, and engine buffers are zeroized on session drop. That does **not** prove physical erasure of WASM linear memory, JavaScript strings, renderer/DOM, clipboard, or OS/swap copies.

---

## 1. Security architecture

```text
                    ┌─────────────────────────────────────┐
                    │           USER ACTIONS                │
                    │  create / unlock / send / swap / ✕   │
                    └──────────────────┬──────────────────┘
                                       │
          ┌────────────────────────────┼────────────────────────────┐
          ▼                            ▼                            ▼
   ┌──────────────┐           ┌────────────────┐           ┌─────────────────┐
   │ Password UI  │           │  Product UI    │           │ Remove account  │
   │ setup/unlock │           │  send/swap/…   │           │ ✕ confirm only  │
   └──────┬───────┘           └───────┬────────┘           └────────┬────────┘
          │                           │                             │
          │ password bytes from input │ sign / reveal               │ strip + drop
          │ (field cleared after use) │                             │ vault entry
          ▼                           ▼                             ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │                         VAULT ENGINE                                      │
   │  JavaScript (legacy): AES-256-GCM + PBKDF2 · JIT decrypt → sign → purge  │
   │  Rust/WASM (migrated): JS holds a WASM handle; WASM holds session         │
   │  material sufficient to sign while unlocked. Same password · addresses.   │
   │  Legacy JS ciphertext retained as recovery material (no routine fallback).│
   └──────────────────────────────┬───────────────────────────────────────────┘
                                  │
          ┌───────────────────────┼────────────────────────┐
          ▼                       ▼                        ▼
   chrome.storage.local     engine-store blob        chrome.storage.session
   public accounts +        rust record +            JS-vault session password
   JS vault ciphertext      legacy JS ciphertext     (not used when Rust-active)
          │
          ▼
   Service worker: dApp routing, auto-lock deadline.
   JS-vault only: short-lived signer cache. Rust-active: no JS secrets in SW.
   Ledger path (parallel): keys NEVER enter the software vault — HID device only.
```

### 1.1 Design goals (always encrypted at rest)

1. **Disk / profile storage** holds only **encrypted** key material (plus public addresses and non-secret settings) — **whether software password is ON or OFF**.
2. **Plain private keys / seed** are **not** kept on account objects while browsing.
3. Required secret material is used **just-in-time** for **sign / approve / intentional seed reveal**, then dropped from application-controlled buffers.
4. **First install** does not generate a wallet until password create (**password-first**).
5. Encryption after password create is **automatic** (no separate “Encrypt” button).
6. **Password ON:** session is unlocked or locked (must log in again after lock / auto-lock).
7. **Password OFF (JavaScript vault):** seeds stay **encrypted** with a **device wrap key**; no password prompt. **Rust-active password-OFF is not claimed to work the same way** until that exact path is tested.
8. **Deleting a seed** only via **✕ Remove** on that account (explicit confirm) — never when toggling password.
9. **Rust-active** wallets do **not** keep the password as a JavaScript string and do **not** fall back to JavaScript vault decrypt for routine signing.

### 1.2 Two engines (same password, same addresses)

| | **JavaScript vault** | **Rust/WASM vault** |
|--|----------------------|---------------------|
| When | Default until a verified migrate | After a successful, verified migrate |
| Cipher | AES-256-GCM | AES-256-GCM (Rust record) |
| KDF | PBKDF2-HMAC-SHA-256, **650,000** iterations | Same class of password-based wrap |
| Unlocked session in JS | Session password string + session store | **WASM handle only** (no password, no private keys) |
| Unlocked session in WASM | n/a | Encrypted vault/session material sufficient to authorize operations. Does **not** keep every private key continuously decrypted. |
| Signing | JIT decrypt in JS → sign → purge | Required key material decrypted/derived in WASM → sign → application-controlled buffers zeroized. Signature/signed tx only returns to JS. |
| Reveal | Decrypt payload, show requested field | One-field export after password step-up. **Exception:** that one secret temporarily becomes a JS string and DOM text. |
| Password in JS | Held while unlocked (needed to open the JS blob) | Ingested as bytes from the login field, then wiped |
| Legacy ciphertext | The live JS blob | **Retained as recovery material.** No routine or automatic fallback to it is permitted while Rust is active. |

Migration does **not** change the seed, password, or addresses. Packaged runtime stays on the JavaScript owner until an unpacked, operator-armed migrate succeeds. Settings does not offer a routine rollback while the Rust pointer is active.

### 1.3 Crypto parameters (software vault)

| Parameter | Value |
|-----------|--------|
| Cipher | AES-256-GCM |
| KDF | PBKDF2-HMAC-SHA-256 |
| Iterations | **650,000** current (stored `iterations` used on decrypt; older 310k / 16-byte salt still open, then upgraded) |
| Salt | 32-byte random (per vault) |
| IV | Random per encryption |
| JS payload shape | `{ v: 1, secrets: { [accountId]: { mnemonic, solanaSecretKey, evmPrivateKey, bitcoinPrivateKey, suiSecretKey } } }` |
| Wrap modes | `user` (password) · `device` (`wrapMode: "device"`) — **JS vault.** Rust-active device-wrap is unverified. |

### 1.4 What lives where

| Name / store | Role | Plain seed? |
|--------------|------|-------------|
| Public `accounts[]` | Name, addresses, type | **No** after strip |
| JS vault blob (`smart_wallet_v1`) | Ciphertext | **No** |
| Engine pointer | `javascript` or `rust` | — |
| Engine store | Rust encrypted record + legacy JS ciphertext | **No** |
| `vaultEnabled` | `true` = user password; `false` = device wrap (JS path) | — |
| JS session password | Opens the **JS** vault while unlocked | Not seed; **unused when Rust-active** |
| Device wrap key | Random wrap secret when password OFF (**JS path**) | **Not** the seed |
| WASM session handle (JS) | Rust-active liveness token | Handle only |
| WASM session (inside WASM) | Encrypted/session material sufficient to sign while unlocked | Required secret material only, temporarily |
| Auto-lock activity row | `lastActivityAt` + `autoLockDeadlineAt` (Unix ms) | No secrets |
| SW signer cache | JS-vault warm signer while unlocked | **Yes briefly** (JS engine only) |

---

## 2. Password ON vs OFF

| | **Password ON** | **Password OFF** |
|--|-----------------|------------------|
| Keys on disk (JS vault) | Encrypted with **user password** | Encrypted with **device wrap key** |
| Seed long-term in browser | Never plain | Never plain |
| Required secret material in RAM | JIT sign / approve / reveal / gen / import | JS path: same. **Rust-active: not verified** |
| Unlock | Type password after lock / auto-lock | JS path: no password prompt |
| Rust-active unlock | Password bytes into WASM, then wiped | **Do not assume device wrap of the Rust record** |
| Same wallets/seeds | Yes | JS path: yes (re-wrapped, not regenerated) |
| Remove a seed | Only **✕** on account | Same |

On the **JavaScript vault**, turning password **off** re-encrypts the same secrets under the device wrap key after confirm. It does **not** write plaintext seeds and does **not** delete wallets.

That paragraph describes the JS path. **An already migrated Rust vault has not been proven to behave identically if Password is turned off.** Do not treat device wrap as the Rust-active OFF model until that exact path is tested.

---

## 3. Locked vs unlocked

Private keys stay ciphertext on disk in both states.

| | **Locked** | **Unlocked (JS vault)** | **Unlocked (Rust/WASM)** |
|--|------------|-------------------------|--------------------------|
| Seed on disk | Encrypted | Encrypted | Encrypted |
| Plain keys on accounts while browsing | **No** | **No** | **No** |
| Session password in JS | **No** | **Yes** | **No** |
| What JS holds | Held lock | Session password | WASM **handle** only |
| What WASM holds | Drained; application-controlled buffers zeroized | n/a | Encrypted/session material sufficient to authorize operations (not every key continuously decrypted) |
| Can sign without typing password again | **No** | **Yes** (JIT in JS) | **Yes** (WASM sign), if the handle is still live |

**Unlocked does not mean “keys sit plain in the wallet.”**

**Auto-lock** uses one stored deadline: `autoLockDeadlineAt = lastQualifyingActivity + selectedDuration` (Unix milliseconds). Selectable durations: **15 minutes, 30 minutes, 1 hour, 2 hours, 3 hours, 12 hours, 24 hours, 3 days**. Closing the popup does not count as expiry. Manual **Lock** is immediate. Wrong-password lockout (10 failures → 1 hour) is separate.

**12 hours, not 112 hours.** A 112-hour Settings option was a typo and is **not** offered. If an older session stored 6720 minutes, that value is treated as 12 hours.

Rust-active reopen after popup close usually **cannot** silent-restore: the WASM handle is gone and no JS password is kept. That is fail-closed login, not timeout expiration.

---

## 4. Key lifecycle

### Phase A — Before the key exists

No mnemonic is generated until password create succeeds.

### Phase B — Birth (first plaintext window, JS create path)

| Step | What happens | Required secret material? |
|------|----------------|---------------------------|
| B1 | User sets + confirms password | No |
| B2 | 32-byte entropy → BIP39 phrase | **Yes — mnemonic** |
| B3 | Derive Solana / EVM / BTC / Sui keys | **Yes — mnemonic + chain keys** |
| B4 | Encrypt into the vault | **Yes until encrypt completes** |
| B5 | Only **public** account fields kept in UI state | **No** on accounts |
| B6 | Disk write: ciphertext + stripped accounts | **No** on disk |

On a **Rust-active** wallet, later create/import/remove goes through Rust mutate rather than repeating this JS birth table.

### Phase C — Normal life

Home / balances / quotes / history / settings browse: **no** private keys in JS. Rust-active: WASM handle may be live; keys are not continuously decrypted.

### Phase D — Signing

**JavaScript vault**

```text
JIT window
  → decrypt vault with the session password
  → apply secrets → sign → purge
  → may push short-lived secrets to the service-worker signer cache
```

**Rust/WASM vault**

```text
WASM handle already live in this page
  → required key material decrypted or derived inside WASM
  → sign
  → application-controlled buffers zeroized
  → signature / signed transaction only returns to JS
  → no JS vault decrypt
```

Popup close drops the WASM handle. Reopen inside the auto-lock window still needs the password unless a JS session password exists (it does not, on Rust-active).

| User action | Secret material in JS? | Notes |
|-------------|------------------------|--------|
| Send / Swap / Bridge execute | JS vault: **briefly** (and possibly SW cache). Rust-active: **no** for routine sign | Ledger never loads a software seed. Bitcoin send on Rust-active uses a WASM digest sign, not a JS private key. |
| Swap / Bridge quote | **No** | Public + APIs only |
| dApp / WalletConnect approve | JS vault: **briefly**. Rust-active: WASM sign | In-wallet approval only |
| Reveal seed / one chain key | **Yes** until closed | Rust: one field after step-up; that field is a JS string + DOM text until wiped |

### Phase E — Password mode / migrate

| Event | Secret material in JS? | Disk |
|-------|------------------------|------|
| Password OFF / ON / change (JS vault) | **Yes** briefly during re-wrap | Still ciphertext |
| Password OFF on a **Rust-active** wallet | **Not verified** as device-wrap of the Rust record | Do not claim identical behavior |
| Password change on Rust | Rust `changePassword` plus leftover JS blob open | Still ciphertext; JS blob may still be touched |
| Migrate JS → Rust | Password used to open JS blob and write Rust record | Legacy JS ciphertext **kept as recovery material** |
| Rollback Rust → JS | Not a routine path while Rust is active | Pointer would have to be changed deliberately; no automatic fallback |

Seeds are **not** deleted when toggling password or migrating.

### Phase F — Lock / auto-lock

| Event | Effect |
|-------|--------|
| **Lock now** | Mark locked; bump session generation; abort privileged work; drain WASM (application-controlled buffers zeroized); clear JS session password; strip accounts |
| **Auto-lock** | Same teardown when `now >= autoLockDeadlineAt` |
| Restore failure before deadline | Fail closed (login), **not** labeled as timeout |

### Phase G — Destruction

| Event | What happens |
|-------|----------------|
| **✕ Remove account** | That seed removed from the vault map; remaining secrets resealed |
| **Uninstall / clear site data** | Profile storage gone |
| **Forget password** | Vault unreadable; still ciphertext on disk until remove/uninstall |

### Phase H — Ledger

Ledger connect / sign: **no** software seed for that account. Software wallets in the same install still follow Phases B–G.

---

## 5. End-to-end inventory (software)

| # | Moment | In JavaScript? | How long |
|---|--------|----------------|----------|
| 1 | First create (JS path) | Required secret material (mnemonic + derived keys) | Create call |
| 2 | Idle browse | **No** private keys | — |
| 3 | Locked | **No** | Until unlock |
| 4 | Unlocked, not signing (JS vault) | Session password maybe; no keys on accounts | Until lock |
| 5 | Unlocked, not signing (Rust-active) | WASM **handle** only | Until lock or popup teardown |
| 6 | Send / swap / bridge / dApp sign (JS vault) | JIT secrets; may enter SW signer cache | Operation only |
| 7 | Same actions (Rust-active) | Signature / signed tx only | Operation only |
| 8 | Reveal seed / key | **Yes** — selected secret as string + DOM | Until closed / wiped |
| 9 | Password toggle / migrate (JS path) | **Yes** briefly | Operation only |
| 10 | ✕ Remove / uninstall | Destroyed or abandoned | — |

### Sensitive but not a seed

| Item | Notes |
|------|--------|
| JS session password / device wrap | Can open the **JS** vault; **not** the seed |
| WASM session handle | JS liveness token; does not export keys |
| WASM session material | Sufficient to authorize operations while unlocked |
| Public addresses | Safe to show |
| Vault ciphertext (JS and Rust records) | Needs password (+ offline cost of PBKDF2) |

---

## 6. Service worker & multi-surface behavior

| Behavior | Purpose |
|----------|---------|
| JS-vault: push secrets to SW while unlocked | External DEX / dApp can sign if popup closed within auto-lock |
| Rust-active: SW does not receive JS secrets | Signing stays in WASM |
| Session unlock blob | JS-vault only; survives MV3 SW restart inside auto-lock |
| Canonical auto-lock deadline | Shared across popup, full-page wallet, and service worker |
| Lock | Clear warm signer, drain WASM (zeroize application-controlled buffers), persist held lock |

RPC, price, quote and portfolio modules are not intentionally given private keys.

---

## 7. Threat surfaces (honest limits)

### 7.1 RAM

During create, import, JS-vault sign, reveal, and re-wrap, required secret material **is** in process memory. Rust/WASM **shortens** the JavaScript window for routine sign (no password string, no decrypted key map).

On session drop, application-controlled Rust password, blob, and engine buffers are **zeroized**. Smart Wallet still cannot prove physical erasure of WASM linear memory after drop, immutable JavaScript strings, renderer/DOM, clipboard, or OS/swap.

### 7.2 Persistent storage

| Store | Intended content |
|-------|------------------|
| `chrome.storage.local` | Stripped accounts + vault ciphertext (+ engine store if migrated) |
| Device wrap key (password OFF, **JS path**) | Random secret — weaker than a strong user password against full profile theft |

### 7.3 Ledger

Keys never leave the device for pure Ledger accounts.

---

## 8. Lifecycle checklist (operator view)

1. **Install empty** → force password create.
2. **Password create** → auto generate + encrypt + public-only state.
3. **Password ON + unlocked** → browse public data; keys ciphertext on disk.
4. **Optional migrate to Rust/WASM** (unpacked, operator-armed) → same password and addresses; legacy JS blob kept as recovery material; no routine fallback while Rust is active.
5. **Sign / approve** → JS vault: brief required secret material → purge (SW cache possible). Rust-active: WASM sign; JS gets signature only.
6. **Reveal** → selected secret temporarily in JS + DOM.
7. **Lock / auto-lock** → session cleared; WASM drained and application-controlled buffers zeroized; disk still ciphertext.
8. **Password OFF** → JS path re-wraps with a device key. Rust-active OFF is **unverified**.
9. **✕ Remove** → discard that wallet’s secrets.
10. **Uninstall / clear data** → abandon remaining ciphertext.

---

## 9. What Smart Wallet does **not** claim

- Keys never exist in RAM
- Perfect zeroization of every temporary buffer
- Physical erasure of WASM linear memory, JS strings, DOM, clipboard, or swap
- Password never in memory while a **JavaScript** vault is unlocked
- Device wrap as strong as a high-entropy user password against full disk/profile theft
- Identical Password-OFF / device-wrap behavior on an already migrated Rust vault (untested)
- Routine user rollback from Rust to JS while the Rust pointer is active
- Silent Rust unlock after popup close without typing the password
- Protection against malware dumping the process during sign or first create
- Same guarantees as a hardware wallet

**High-value funds:** prefer **Ledger**, OS disk encryption (e.g. BitLocker), clean machine, **password ON** + strong password, offline seed backup.

---

## 10. Mapping (for auditors)

| Concern | Behavior |
|---------|----------|
| Create mnemonic + derive chains | Entropy → BIP39 → HD derive → encrypt (JS birth path) |
| JS encrypt / decrypt | AES-256-GCM + PBKDF2-HMAC-SHA-256 |
| JS strip / purge | Accounts hold public fields only after JIT |
| JS JIT window | Decrypt for the call, purge in `finally`; may warm SW signer cache |
| Rust unlock | Password bytes from the login field → WASM → wipe bytes; JS keeps handle |
| Rust session | Encrypted/session material in WASM, sufficient to authorize while unlocked |
| Rust sign | Required secret material in WASM RAM temporarily; signature/signed tx out; application-controlled buffers zeroized |
| Rust reveal | One field after step-up; that field is a JS string + DOM until wiped |
| Lock | Held lock + session generation + WASM drain + zeroize |
| Auto-lock | Stored Unix-ms deadline; options 15m–3d including **12 hours** (not 112 hours) |
| Wrong password | 10 failures → 1 hour; Rust counts authentication failure only |
| Destroy account | ✕ Remove + reseal remaining secrets |

---

*Not financial advice. Cryptocurrency involves risk of loss. This is a product architecture description, not a formal security audit.*
