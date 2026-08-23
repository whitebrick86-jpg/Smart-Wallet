# Key protection in Smart Wallet

**Product:** Smart Wallet (Chrome / Opera MV3 browser extension)  
**Repo:** [whitebrick86-jpg/Smart-Wallet](https://github.com/whitebrick86-jpg/Smart-Wallet) (docs only)  
**Code snapshot:** **0.11.473** (Load-unpacked extension; source not published here)  
**Primary storage:** `smart_wallet_v1` — public account metadata + **encrypted vault**  
**Last updated:** 2026-08-23

This document describes **how software keys and passwords are protected** in the current product. It covers the **JavaScript vault** (legacy / unmigrated) and the **Rust/WASM vault** (after the user confirms migration). Networking modules **never** hold private keys.

**Related:** [ARCHITECTURE.md](./ARCHITECTURE.md) (RPC, managers, tx lifecycle).

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
   │  Rust/WASM (migrated): WASM session handle · sign/reveal inside WASM     │
   │  Same user password · same addresses · legacy JS ciphertext kept         │
   └──────────────────────────────┬───────────────────────────────────────────┘
                                  │
          ┌───────────────────────┼────────────────────────┐
          ▼                       ▼                        ▼
   chrome.storage.local     engine-store blob        chrome.storage.session
   public accounts +        rust record +            JS-vault session password
   JS vault ciphertext      legacy JS ciphertext     (not used when Rust-active)
          │
          ▼
   Service worker: dApp routing, auto-lock deadline, no long-term seeds
   Ledger path (parallel): keys NEVER enter the software vault — HID device only.
```

### 1.1 Design goals (always encrypted at rest)

1. **Disk / profile storage** holds only **encrypted** key material (plus public addresses and non-secret settings) — **whether software password is ON or OFF**.
2. **Plain private keys / seed** are **not** kept on account objects while browsing.
3. Plain keys are used **just-in-time** for **sign / approve / intentional seed reveal**, then dropped.
4. **First install** does not generate a wallet until password create (**password-first**).
5. Encryption after password create is **automatic** (no separate “Encrypt” button).
6. **Password ON:** session is unlocked or locked (must log in again after lock / auto-lock).
7. **Password OFF:** seeds stay **encrypted** with a **device wrap key**; no password prompt.
8. **Deleting a seed** only via **✕ Remove** on that account (explicit confirm) — never when toggling password.
9. **Rust-active** wallets do **not** keep the password as a JavaScript string and do **not** fall back to JavaScript vault decrypt for signing.

### 1.2 Two engines (same password, same addresses)

| | **JavaScript vault** | **Rust/WASM vault** |
|--|----------------------|---------------------|
| When | Default until the user confirms migration | After a successful, verified migrate |
| Cipher | AES-256-GCM | AES-256-GCM (Rust record) |
| KDF | PBKDF2-HMAC-SHA-256, **650,000** iterations | Same class of password-based wrap |
| Unlocked session | `SESSION_VAULT_PASSWORD` string + session store | **WASM handle** (`wasmHandle`) only |
| Signing | JIT decrypt in JS → sign → purge | Bytes in / signature bytes out inside WASM |
| Reveal | Decrypt payload, show requested field | One-field `export` after password step-up |
| Password in JS | Held while unlocked (needed to open the JS blob) | Ingested as bytes from the login field, then wiped |
| Legacy ciphertext | The live JS blob | **Kept** for rollback; not used for sign |

Migration does **not** change the seed, password, or addresses. The current encrypted JavaScript vault is retained so the user can roll back.

### 1.3 Crypto parameters (software vault)

| Parameter | Value |
|-----------|--------|
| Cipher | AES-256-GCM |
| KDF | PBKDF2-HMAC-SHA-256 |
| Iterations | **650,000** current (stored `iterations` used on decrypt; older 310k / 16-byte salt still open, then upgraded) |
| Salt | 32-byte random (per vault) |
| IV | Random per encryption |
| JS payload shape | `{ v: 1, secrets: { [accountId]: { mnemonic, solanaSecretKey, evmPrivateKey, bitcoinPrivateKey, suiSecretKey } } }` |
| Wrap modes | `user` (password) · `device` (`wrapMode: "device"`) |

### 1.4 What lives where

| Name / store | Role | Plain seed? |
|--------------|------|-------------|
| Public `accounts[]` | Name, addresses, type | **No** after strip |
| JS vault blob (`smart_wallet_v1`) | Ciphertext | **No** |
| Engine pointer | `javascript` or `rust` | — |
| Engine store | Rust encrypted record + legacy JS ciphertext | **No** |
| `vaultEnabled` | `true` = user password; `false` = device wrap | — |
| JS session password | Opens the **JS** vault while unlocked | Not seed; **unused when Rust-active** |
| Device wrap key | Random wrap secret when password OFF | **Not** the seed |
| WASM session handle | Rust-active liveness | Handle only; secrets stay in WASM |
| Auto-lock activity row | `lastActivityAt` + `autoLockDeadlineAt` (Unix ms) | No secrets |
| SW signer cache | JS-vault warm signer while unlocked | **Yes briefly** (JS engine only) |

---

## 2. Password ON vs OFF

| | **Password ON** | **Password OFF** |
|--|-----------------|------------------|
| Keys on disk | Encrypted with **user password** | Encrypted with **device wrap key** |
| Seed long-term in browser | Never plain | Never plain |
| Plain seed in RAM | Only JIT sign / approve / reveal / gen / import | Same |
| Unlock | Type password after lock / auto-lock | No password prompt |
| Rust-active unlock | Password bytes into WASM, then wiped | Device wrap still used by the JS path if not migrated |
| Same wallets/seeds | Yes | Yes (re-wrapped, not regenerated) |
| Remove a seed | Only **✕** on account | Same |

Turning password **off** re-encrypts the same secrets under the device wrap key after confirm. It does **not** write plaintext seeds and does **not** delete wallets.

---

## 3. Locked vs unlocked

Private keys stay ciphertext on disk in both states.

| | **Locked** | **Unlocked (JS vault)** | **Unlocked (Rust/WASM)** |
|--|------------|-------------------------|--------------------------|
| Seed on disk | Encrypted | Encrypted | Encrypted |
| Plain keys on accounts while browsing | **No** | **No** | **No** |
| Session password in JS | **No** | **Yes** | **No** (handle only) |
| WASM session | Drained / zeroized | n/a | **Live handle** |
| Can sign without typing password again | **No** | **Yes** (JIT) | **Yes** (WASM sign) |

**Unlocked does not mean “keys sit plain in the wallet.”**  
It means the session can open the vault for the next sign — via a JS password string (legacy) or a live WASM handle (Rust-active).

**Auto-lock** uses one stored deadline: `autoLockDeadlineAt = lastQualifyingActivity + selectedDuration` (Unix milliseconds). Closing the popup does not count as expiry. Manual **Lock** is immediate. Wrong-password lockout (10 failures → 1 hour) is separate.

---

## 4. First iteration of a key: plaintext from birth to death

### Phase A — Before the key exists

No mnemonic is generated until password create succeeds.

### Phase B — Birth (first plaintext window)

| Step | What happens | Plain keys? |
|------|----------------|-------------|
| B1 | User sets + confirms password | No |
| B2 | 32-byte entropy → BIP39 phrase | **Yes — mnemonic** |
| B3 | Derive Solana / EVM / BTC / Sui keys | **Yes — mnemonic + chain keys** |
| B4 | Encrypt into the vault | **Yes until encrypt completes** |
| B5 | Only **public** account fields kept in UI state | **No** on accounts |
| B6 | Disk write: ciphertext + stripped accounts | **No** on disk |

That is the **first iteration** of the key in plaintext. It should not be re-read from disk in plain form afterward.

### Phase C — Normal life

Home / balances / quotes / history / settings browse: **no** plain keys.

### Phase D — Signing

**JavaScript vault**

```text
JIT window
  → decrypt vault with the session password
  → apply secrets → sign → purge
```

**Rust/WASM vault**

```text
WASM session already live (or restore if still inside auto-lock)
  → sign(account, chain, message bytes) inside WASM
  → signature bytes only return to JS
  → no JS vault decrypt
```

| User action | Plain keys in JS? | Notes |
|-------------|-------------------|--------|
| Send / Swap / Bridge execute | JS vault: **briefly**. Rust-active: **no** (WASM only) | Ledger never loads a software seed |
| Swap / Bridge quote | **No** | Public + APIs only |
| dApp / WalletConnect approve | JS vault: **briefly**. Rust-active: WASM sign | In-wallet approval only |
| Reveal seed / one chain key | **Yes** in UI until closed | Rust: one field, password step-up; empty stored chain key is derived inside WASM |

### Phase E — Password mode / migrate

| Event | Plain keys in JS? | Disk |
|-------|-------------------|------|
| Password OFF / ON / change | **Yes** briefly during re-wrap (JS path) | Still ciphertext |
| Migrate JS → Rust | Password used to open JS blob and write Rust record | Legacy JS ciphertext **kept** |
| Rollback Rust → JS | Uses retained legacy ciphertext | Pointer returns to JavaScript |

Seeds are **not** deleted when toggling password or migrating.

### Phase F — Lock / auto-lock

| Event | Effect |
|-------|--------|
| **Lock now** | Mark locked; bump session generation; abort privileged work; drain WASM; clear JS session password; strip accounts |
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

## 5. End-to-end plaintext inventory (software)

| # | Moment | Plain keys in JS? | How long |
|---|--------|-------------------|----------|
| 1 | First create | **Yes** | Create call |
| 2 | Idle browse | **No** | — |
| 3 | Locked | **No** | Until unlock |
| 4 | Unlocked, not signing (JS vault) | No keys on accounts; session password maybe | Until lock |
| 5 | Unlocked, not signing (Rust-active) | **No** password string; WASM handle only | Until lock |
| 6 | Send / swap / bridge / dApp sign (JS vault) | **Yes** JIT | Operation only |
| 7 | Same actions (Rust-active) | **No** (signature bytes only) | Operation only |
| 8 | Reveal seed / key | **Yes** in UI | Until closed |
| 9 | Password toggle / migrate | **Yes** briefly on JS path | Operation only |
| 10 | ✕ Remove / uninstall | Destroyed or abandoned | — |

### Not plain key material (still sensitive)

| Item | Notes |
|------|--------|
| JS session password / device wrap | Can open the **JS** vault; **not** the seed |
| WASM session handle | Proves the Rust vault is open; does not export keys |
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
| Lock | Clear warm signer, drain WASM, persist held lock |

Transaction managers / RPC gateway / portfolio modules **do not** receive seeds.

---

## 7. Threat surfaces (honest limits)

### 7.1 RAM

During create, import, JS-vault sign, reveal, and re-wrap, plaintext **is** in process memory. Rust/WASM **shortens** the JavaScript window (no password string, no decrypted key map for routine sign). Smart Wallet cannot make software keys **forensically invisible**. Immutable JavaScript strings, renderer/DOM, clipboard, and WASM linear memory after drop cannot be proven physically erased.

### 7.2 Persistent storage

| Store | Intended content |
|-------|------------------|
| `chrome.storage.local` | Stripped accounts + vault ciphertext (+ engine store if migrated) |
| Device wrap key (password OFF) | Random secret — weaker than a strong user password against full profile theft |

### 7.3 Ledger

Keys never leave the device for pure Ledger accounts.

---

## 8. Lifecycle checklist (operator view)

1. **Install empty** → force password create.
2. **Password create** → auto generate + encrypt + public-only state.
3. **Password ON + unlocked** → browse public data; keys ciphertext on disk.
4. **Optional migrate to Rust/WASM** → same password and addresses; legacy JS blob kept.
5. **Sign / approve / reveal** → JS vault: brief plain → purge. Rust-active: WASM sign / one-field reveal.
6. **Lock / auto-lock** → session cleared; WASM drained; disk still ciphertext.
7. **Password OFF** → re-wrap with device key (JS path).
8. **✕ Remove** → discard that wallet’s secrets.
9. **Uninstall / clear data** → abandon remaining ciphertext.

---

## 9. What Smart Wallet does **not** claim

- Keys never exist in RAM
- Perfect zeroization of every temporary buffer
- Password never in memory while a **JavaScript** vault is unlocked
- Device wrap as strong as a high-entropy user password against full disk/profile theft
- Protection against malware dumping the process during sign or first create
- Same guarantees as a hardware wallet
- WASM memory is physically erased the instant a session is dropped

**High-value funds:** prefer **Ledger**, OS disk encryption (e.g. BitLocker), clean machine, **password ON** + strong password, offline seed backup.

---

## 10. Mapping (for auditors)

| Concern | Behavior |
|---------|----------|
| Create mnemonic + derive chains | Entropy → BIP39 → HD derive → encrypt |
| JS encrypt / decrypt | AES-256-GCM + PBKDF2-HMAC-SHA-256 |
| JS strip / purge | Accounts hold public fields only after JIT |
| JS JIT window | Decrypt for the call, purge in `finally` |
| Rust unlock | Password bytes from the login field → WASM → wipe bytes |
| Rust sign | Message bytes in, signature bytes out |
| Rust reveal | One field after step-up; derive-on-empty inside WASM |
| Lock | Held lock + session generation + WASM drain |
| Auto-lock | Stored Unix-ms deadline, not `setTimeout` as source of truth |
| Wrong password | 10 failures → 1 hour; Rust counts authentication failure only |
| Destroy account | ✕ Remove + reseal remaining secrets |

---

*Not financial advice. Cryptocurrency involves risk of loss. This is a product architecture description, not a formal security audit.*
