# Key protection in Smart Wallet

**Product:** Smart Wallet (Chrome / Opera MV3 browser extension)  
**Repo:** [whitebrick86-jpg/Smart-Wallet](https://github.com/whitebrick86-jpg/Smart-Wallet) (docs only)  
**Code snapshot:** **0.11.36+** (Load-unpacked extension; source not published here)  
**Primary storage:** `smart_wallet_v1` â€” public account metadata + **encrypted vault**  
**Last updated:** 2026-08-11  

This document describes **how software keys and passwords are protected**, mapped to the current implementation (`app.js` vault layer + service worker session/signer). It includes the **full lifetime of a keyâ€™s first plaintext appearance** â€” from generation through password ON/OFF, signing, and destruction.

**Related:** [ARCHITECTURE.md](./ARCHITECTURE.md) (RPC, managers, tx lifecycle). Networking modules **never** hold private keys.

---

## 1. Security architecture (how the key system is built)

```text
                    â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
                    â”‚           USER ACTIONS                â”‚
                    â”‚  create / unlock / send / swap / âœ•   â”‚
                    â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                                       â”‚
          â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¼â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
          â–¼                            â–¼                            â–¼
   â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”           â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”           â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
   â”‚ Password UI  â”‚           â”‚  Product UI    â”‚           â”‚ Remove account  â”‚
   â”‚ setup/unlock â”‚           â”‚  send/swap/â€¦   â”‚           â”‚ âœ• confirm only  â”‚
   â””â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”˜           â””â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”˜           â””â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”˜
          â”‚                           â”‚                             â”‚
          â”‚ SESSION_VAULT_PASSWORD    â”‚ withEphemeralSoftwareSecrets â”‚ strip + drop
          â”‚ (user or device wrap)     â”‚   JIT decrypt â†’ sign â†’ purge â”‚ vault entry
          â–¼                           â–¼                             â–¼
   â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
   â”‚                         VAULT LAYER (app.js)                              â”‚
   â”‚  extractSecretsMap / encryptVaultPayload / decryptVaultPayload          â”‚
   â”‚  stripAccountSecrets / purgeSoftwareSecretsFromMemory                   â”‚
   â”‚  AES-256-GCM  Â·  PBKDF2-HMAC-SHA-256 Â· 650k iterations Â· 32-byte salt   â”‚
   â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                                   â”‚
          â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¼â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
          â–¼                        â–¼                        â–¼
   chrome.storage.local      localStorage mirror      chrome.storage.session
   smart_wallet_v1           (stripped + vault)       session unlock blob +
   (ciphertext vault)                                  session password
          â”‚
          â”‚ pushWalletStateToServiceWorker(allowSecrets)
          â–¼
   â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
   â”‚  SERVICE WORKER (background.js)          â”‚
   â”‚  warm signer cache (while unlocked)      â”‚
   â”‚  dApp / external DEX sign within lock    â”‚
   â”‚  clear caches on lock / stripped push    â”‚
   â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜

   Ledger path (parallel): keys NEVER enter software vault â€” HID device only.
```

### 1.1 Design goals (always encrypted at rest)

1. **Disk / profile storage** holds only **encrypted** key material (plus public addresses and non-secret settings) â€” **whether software password is ON or OFF**.  
2. **Plain private keys / seed** are **not** kept on `STATE.accounts` while browsing.  
3. Plain keys are loaded **just-in-time (JIT)** for **sign / approve / intentional seed reveal**, then purged.  
4. **First install** does not generate a wallet until password create (**password-first**).  
5. Encryption after password create is **automatic** (no separate â€œEncryptâ€ button).  
6. **Password ON:** session is unlocked (password available) or locked (must log in again).  
7. **Password OFF:** seeds stay **encrypted** with a **device wrap key**; no password prompt; JIT decrypt still required for sign.  
8. **Deleting a seed** only via **âœ• Remove** on that account (explicit confirm) â€” never when toggling password.

### 1.2 Crypto parameters (software vault)

| Parameter | Value |
|-----------|--------|
| Cipher | AES-256-GCM |
| KDF | PBKDF2-HMAC-SHA-256 |
| Iterations | **650,000** (`VAULT_KDF_ITERATIONS`) |
| Salt | 32-byte random (per vault) |
| IV | Random per encryption |
| Payload shape | `{ v: 1, secrets: { [accountId]: { mnemonic, solanaSecretKey, evmPrivateKey, bitcoinPrivateKey, suiSecretKey } } }` |
| Wrap modes | `user` (password) Â· `device` (`wrapMode: "device"`) |

### 1.3 What lives where (variables)

| Name / store | Role | Plain seed? |
|--------------|------|-------------|
| `STATE.accounts[]` | Public metadata (name, addresses, type) | **No** after strip |
| `STATE.vault` | Ciphertext blob (`data`, `salt`, `iv`, `iterations`) | **No** |
| `STATE.vaultEnabled` | `true` = user password mode; `false` = device wrap mode | â€” |
| `SESSION_VAULT_PASSWORD` | In-process password or device wrap string while usable | Not seed; can open vault |
| Session store (`smart_wallet_session_pw_v1` etc.) | Survives popup close within browser session / auto-lock window | Password/wrap only |
| `smart_wallet_device_wrap_v1` | Random **device wrap** secret when password OFF | **Not** the seed |
| `smart_wallet_v1` | Canonical persisted wallet | Ciphertext + stripped accounts |
| SW `cachedSigner` / `cachedSecretsMap` | Warm signer while unlocked (dApp / Jupiter) | **Yes briefly** while warm |
| `SECRETS_EPHEMERAL_DEPTH` | Nesting counter for JIT windows | Controls when purge runs |

---

## 2. Password ON vs OFF

| | **Password ON** (`vaultEnabled: true`) | **Password OFF** (`vaultEnabled: false`) |
|--|----------------------------------------|------------------------------------------|
| Keys on disk | Encrypted with **user password** | Encrypted with **device wrap key** |
| Seed long-term in browser | Never plain | Never plain |
| Plain seed in RAM | Only JIT sign / approve / reveal / gen / import | Same |
| Unlock | Type password after lock / auto-lock | No password prompt |
| Session password | User string | Device wrap auto-loaded into session |
| Same wallets/seeds | Yes | Yes (re-wrapped, not regenerated) |
| Remove a seed | Only **âœ•** on account | Same |

Turning password **off** re-encrypts the same secrets under the device wrap key after confirm (*Are you sureâ€¦?*). It does **not** write plaintext seeds and does **not** delete wallets.

Turning password **on** re-encrypts under a user password; seeds are not re-derived from scratch.

---

## 3. Locked vs unlocked (user password mode)

Under always-encrypt-at-rest, **private keys stay ciphertext on disk in both states**. The difference is whether the next software sign can decrypt **without re-entering the password**.

| | **Locked** | **Unlocked** |
|--|------------|--------------|
| Seed / private keys on disk | Encrypted vault | Encrypted vault |
| Plain keys on account objects while browsing | **No** | **No** |
| Session password in memory | **No** (cleared) | **Yes** (after login) |
| SW warm signer | Cleared | May be warm within auto-lock |
| Can sign without typing password again | **No** | **Yes** (JIT decrypt â†’ sign â†’ purge) |

**Unlocked does not mean â€œkeys sit plain in the wallet.â€**  
It means: *the session holds material that can open the vault for the next sign*.

---

## 4. First iteration of a key: plaintext from birth to death

This section follows **one software wallet** (the first account generated after install) and every moment its **plaintext** seed / private keys exist.

### Phase A â€” Before the key exists

| Step | Plain key material? |
|------|---------------------|
| Extension installed, no password | **None** â€” no wallet generated |
| User opens create-password UI | **None** |

Password-first: **no mnemonic is generated until password create succeeds.**

### Phase B â€” Birth (first plaintext window)

| Step | What happens | Plain keys? | Where | Duration |
|------|----------------|-------------|-------|----------|
| B1 | User sets + confirms password | No | â€” | â€” |
| B2 | `createTwentyFourWordMnemonic()` â€” 32-byte entropy â†’ BIP39 phrase | **Yes â€” mnemonic** | Local JS (`phrase`) | Milliseconds |
| B3 | `keysFromMnemonic(phrase, 0)` â€” seed from mnemonic; derive Solana/EVM/BTC/Sui keys | **Yes â€” mnemonic + all chain private keys** | Local object returned by `keysFromMnemonic` / `createAccount` | Same call stack |
| B4 | Build secrets map from that local object | **Yes** | Ephemeral map | Same |
| B5 | `encryptVaultPayload(password, { secrets })` â€” AES-GCM | **Yes until encrypt completes** | Input buffer to encrypt | Millisecondsâ€“seconds |
| B6 | Only **public** account fields written to `STATE.accounts` | **No** on STATE | Addresses, name, id | â€” |
| B7 | Local plain fields scrubbed; map refs dropped | **No** (app references cleared) | â€” | â€” |
| B8 | `storageSet` â†’ disk holds **ciphertext vault + stripped accounts** | **No** on disk | `smart_wallet_v1` | Persistent |

**First plaintext lifetime (create):** entropy â†’ mnemonic â†’ chain keys â†’ encrypt â†’ strip.  
That is the **first iteration** of the key in plaintext. It should not be re-read from disk in plain form afterward.

### Phase C â€” Normal life (encrypted at rest)

| Activity | Plain keys? |
|----------|-------------|
| Home / balances / quotes / history / settings browse | **No** |
| Idle with password ON unlocked | **No** keys on accounts; **session password** may remain |
| Idle locked | **No** keys; **no** session password |
| Password OFF, idle | **No** keys; device wrap may sit in session (not seed) |

### Phase D â€” Signing (recurring plaintext windows)

Each software sign path uses the same pattern:

```text
withEphemeralSoftwareSecrets / materialize + push SW
  â†’ decrypt vault with getActiveVaultPassword()
  â†’ apply secrets onto accounts (or SW cache only)
  â†’ sign transaction
  â†’ purgeSoftwareSecretsFromMemory / strip accounts
```

| # | User action | Plain keys? | Notes |
|---|-------------|-------------|--------|
| D1 | **Send** (execute) | **Yes** briefly | JIT for software; Ledger never loads software seed |
| D2 | **Swap / Bridge quote** | **No** | Public + APIs only |
| D3 | **Swap / Bridge execute** | **Yes** briefly | Same JIT as send |
| D4 | **dApp / WalletConnect approve** | **Yes** briefly | Popup JIT and/or SW warm signer |
| D5 | **Platform fee settle** (after internal swap/bridge) | **Yes** briefly | Same vault; fee ledger is separate (anti double-charge) |
| D6 | **Reveal seed (backup UI)** | **Yes** in UI + ephemeral hydrate | Until panel closed / purged |

**Password ON + locked:** user must unlock first; then D1â€“D5.  
**Password OFF:** no unlock UI; device wrap opens vault for JIT only.

### Phase E â€” Password mode changes (no new seed)

| Event | Plain keys? | Disk result |
|-------|-------------|-------------|
| **OFF â†’** decrypt with user password, re-encrypt with device wrap | **Yes** briefly during re-wrap | Still ciphertext; `vaultEnabled=false` |
| **ON â†’** decrypt with device wrap, re-encrypt with user password | **Yes** briefly | Ciphertext; `vaultEnabled=true` |
| Change password | **Yes** briefly | New ciphertext, same secrets |

Seeds are **not** deleted when toggling password.

### Phase F â€” Lock / auto-lock

| Event | Effect |
|-------|--------|
| **Lock now** (user mode) | Strip accounts; clear `SESSION_VAULT_PASSWORD`; clear session password store; SW drops warm secrets |
| **Auto-lock** | Same idea after idle (`chrome.alarms` + activity stamp) |
| **Password OFF** | Software â€œlockâ€ does not apply the same way; no user password to forget (device wrap remains usable for JIT) |

### Phase G â€” Destruction / abandonment

| Event | What happens to the key |
|-------|-------------------------|
| **âœ• Remove account** (confirm) | That accountâ€™s entry removed from `accounts` and from vault secrets map; reseal remaining secrets; **that seed is abandoned** (not recoverable from Smart Wallet if vault no longer holds it) |
| **Uninstall extension / clear site data** | Profile storage gone â€” keys unrecoverable from the product (user must have written seed elsewhere) |
| **Abandon without remove** (stop using wallet) | Ciphertext may remain on disk until remove/uninstall; treat as **still present encrypted** |
| **Forget password (user mode)** | Vault unreadable without password; seed still encrypted on disk until remove/uninstall â€” **not** plaintext |

There is **no** automatic â€œwipe after inactivityâ€ of the vault ciphertext. **Remove** or **profile wipe** is the deliberate end of life.

### Phase H â€” Ledger accounts (parallel universe)

| Moment | Software seed on PC? |
|--------|----------------------|
| Ledger connect / Link EVM / sign | **No** software seed for that account |
| Same install has software wallets | Those still follow Phases Bâ€“G |

---

## 5. End-to-end plaintext inventory (software)

| # | Moment | Plain keys? | Where | How long |
|---|--------|-------------|-------|----------|
| 1 | First create (password-create path) | **Yes** | Local generate â†’ encrypt only | Create call |
| 2 | Idle browse | **No** | â€” | â€” |
| 3 | Locked (user mode) | **No** keys | Disk ciphertext | Until unlock |
| 4 | Unlocked, not signing | **No** keys on accounts; session password maybe | Session RAM | Until lock |
| 5 | Send / swap execute / bridge submit / dApp sign / fee settle | **Yes** | Ephemeral + optional SW cache | Operation only |
| 6 | Reveal seed | **Yes** | UI + ephemeral | Until closed |
| 7 | Generate/import another account | **Yes** briefly | Create/import â†’ reseal | Operation only |
| 8 | Password ON/OFF toggle / change password | **Yes** briefly | Re-wrap | Operation only |
| 9 | âœ• Remove / uninstall | Destroyed or abandoned | â€” | â€” |

### Not plain key material (still sensitive)

| Item | Notes |
|------|--------|
| Session password / device wrap string | Can open vault; **not** the seed |
| Public addresses | Safe to show |
| Vault ciphertext | Needs password/wrap (+ offline attack cost of PBKDF2) |
| Balances / history | Not private keys |

---

## 6. Service worker & multi-surface behavior

| Behavior | Purpose |
|----------|---------|
| Push secrets to SW when unlocked | External DEX / dApp can sign if popup closed within auto-lock |
| Stripped push after login | Popup UI does not retain plain keys |
| Session unlock blob (`chrome.storage.session`) | Survive MV3 SW restart without re-prompt inside auto-lock window |
| Lock / stripped push | Clear warm signer + session material |

Transaction managers / RPC gateway / portfolio modules **do not** receive seeds.

---

## 7. Threat surfaces (honest limits)

### 7.1 RAM

During create, import, sign, reveal, and re-wrap, plaintext **is** in process memory. Smart Wallet **shortens** the window; it cannot make software keys **forensically invisible**.

### 7.2 Persistent storage

| Store | Intended content |
|-------|------------------|
| `chrome.storage.local` / `localStorage` | Stripped accounts + vault ciphertext |
| Device wrap key (password OFF) | Random secret â€” weaker than a strong user password against full profile theft |

### 7.3 Ledger

Keys never leave the device for pure Ledger accounts â€” correct model if the goal is â€œseed never on PC.â€

---

## 8. Lifecycle checklist (operator view)

1. **Install empty** â†’ force password create.  
2. **Password create** â†’ auto generate + encrypt + public-only state (**first plaintext iteration ends**).  
3. **Password ON + unlocked** â†’ browse public data; session password held; keys ciphertext on disk.  
4. **Sign / approve / reveal / generate / import** â†’ brief plain â†’ purge / reseal.  
5. **Lock / auto-lock** â†’ session cleared; disk still ciphertext.  
6. **Password OFF** â†’ re-wrap with device key; JIT sign without prompt.  
7. **âœ• Remove** â†’ discard that walletâ€™s secrets from vault.  
8. **Uninstall / clear data** â†’ abandon remaining ciphertext.

---

## 9. What Smart Wallet does **not** claim

- Keys never exist in RAM  
- Perfect zeroization of every temporary buffer  
- Password never in memory while unlocked (user mode)  
- Device wrap as strong as a high-entropy user password against full disk/profile theft  
- Protection against malware dumping the process during sign or first create  
- Same guarantees as a hardware wallet  

**High-value funds:** prefer **Ledger**, OS disk encryption (e.g. BitLocker), clean machine, **password ON** + strong password, offline seed backup.

---

## 10. Mapping to code (for auditors / agents)

| Concern | Primary symbols / files |
|---------|-------------------------|
| Create mnemonic + derive chains | `createTwentyFourWordMnemonic`, `keysFromMnemonic`, `createAccount` |
| Encrypt / decrypt | `encryptVaultPayload`, `decryptVaultPayload`, `deriveVaultKey` |
| Strip / purge | `stripAccountSecrets`, `purgeSoftwareSecretsFromMemory`, `extractSecretsMap` |
| JIT window | `withEphemeralSoftwareSecrets`, `getActiveVaultPassword` |
| Unlock / lock | `unlockVaultWithPassword`, `lockWalletNow` |
| Password OFF/ON | `disablePasswordProtection`, `enablePasswordProtection`, `getOrCreateDeviceWrapPassword` |
| Disk write | `buildDiskState`, `storageSet` |
| SW warm signer | `pushWalletStateToServiceWorker`, `cacheSignerFromState`, session unlock blob |
| Destroy account | Account remove (âœ•) + reseal vault |

---

*Not financial advice. Cryptocurrency involves risk of loss. This is a product architecture description, not a formal security audit.*
