# Key protection in Smart Wallet

**Product:** Smart Wallet (Chrome / Opera MV3 browser extension)
**Repository:** Documentation only; extension source is not published here
**Product snapshot:** **0.11.698** (stamp **w41**)
**Software-key owner:** **Rust/WASM vault**
**Last updated:** 2026-09-13

This document describes the current software-key and Ledger protection model. RPC, price, quote, portfolio, messaging and dApp-discovery modules are not intentionally given private keys.

## Core protection model

- Software-wallet secrets are encrypted at rest.
- Rust/WASM owns software-wallet creation, import, unlock, mutation, backup, recovery, reveal and routine signing.
- While unlocked, the wallet interface holds only a WASM session handle rather than the password, seed phrase or private keys. `rustJsVaultForbidden` is **always true**: there is **no plaintext popup `SESSION_VAULT_PASSWORD`** while Rust owns the vault.
- For routine signing, the required secret material is decrypted or derived inside WASM. Only a signature or signed transaction leaves the vault boundary.
- Application-controlled Rust password, blob and engine buffers are zeroized when their operation or session ends.
- Ledger account keys remain on the Ledger device and are never imported into the software vault.
- Public addresses and non-secret settings may be stored and displayed by the extension.

```text
User approval
     |
     v
Rust/WASM vault session ---- encrypted vault record in extension storage
     |
     +---- derive/decrypt only what the approved operation requires
     |
     +---- return signature or signed transaction

Ledger account ------------ sign on the hardware device
```

## At rest

The extension stores encrypted software-wallet material plus public account metadata. A strong user password protects the vault when Password is ON. When Password is OFF, the wallet uses its device-wrap flow so the seed is not stored as plaintext.

Encryption at rest reduces exposure from copied storage, but device wrapping is not equivalent to a strong user password against complete compromise of the browser profile or operating system.

## Locked and unlocked states

| State | Software secrets on disk | What the wallet interface retains | Can routine signing proceed? |
|---|---|---|---|
| Locked | Encrypted | No live vault session | No |
| Unlocked | Encrypted | WASM session handle and public data | Yes, inside WASM |
| Ledger | Not held by the extension | Public address and device connection state | Only after approval on the device |

Manual Lock and auto-lock invalidate privileged work and drain the software-vault session. The wallet uses a shared auto-lock deadline across its extension surfaces. Wrong-password lockout is separate from the chosen auto-lock duration.

## Key lifecycle

1. **Create or import:** secret material exists briefly while Rust/WASM derives the supported chain accounts and seals the encrypted vault record.
2. **Browse:** balances, prices, quotes, history and settings use public information; private keys are not required.
3. **Sign:** the user reviews an operation, required key material is used inside WASM, and the signed result is returned.
4. **Reveal:** revealing a seed or chain key is an explicit exception. The selected secret must cross into the extension page for display and remains sensitive until that view is closed and cleared.
5. **Lock:** the live session is invalidated and application-controlled sensitive buffers are cleared.
6. **Remove account:** the explicit remove action deletes that account's secret material and reseals what remains.
7. **Uninstall or clear extension data:** locally stored encrypted wallet material is removed with the browser profile data.

## Password behavior

| Mode | At-rest protection | Unlock behavior |
|---|---|---|
| Password ON | User-password-protected encrypted vault | Password required after lock |
| Password OFF | Device-wrapped encrypted vault | No routine password prompt; protection depends more heavily on the device and browser profile |

Changing password mode does not create a new wallet or intentionally change its public addresses.

## Signing boundaries

| Operation | Private key intentionally exposed outside the vault? |
|---|---|
| Balance, price, quote or history lookup | No |
| Send, swap, bridge or dApp signature | No; routine software signing occurs inside WASM |
| Ledger signing | No; key remains on Ledger |
| Explicit seed/key reveal | The selected secret is temporarily displayed and must be treated as exposed to the page and screen |

Transaction details still require user review. Secure key custody cannot make an approved malicious transaction safe.

## Honest limits

Smart Wallet does not claim that:

- secret material never exists in process memory;
- every browser, WASM, renderer, clipboard or operating-system copy can be physically erased;
- the wallet can protect keys from a fully compromised operating system while they are being used;
- encryption compensates for a weak password, malicious extension, phishing approval or unsafe backup.

For high-value accounts, prefer Ledger, a strong wallet password, operating-system disk encryption, a clean device and an offline recovery backup.

## Operational invariants

- No plaintext seed or private key should be persisted in account metadata.
- Lock and account changes must invalidate stale signing work.
- External requests and provider responses must be validated before any signing request is presented.
- Routine signing must remain inside Rust/WASM.
- While `rustJsVaultForbidden` is true (always in this product), do not store or mirror a plaintext popup `SESSION_VAULT_PASSWORD`.
- Reveal must remain explicit, narrowly scoped and visibly sensitive.
- Logs and telemetry must never contain passwords, seed phrases or private keys.

---

*This is a product architecture description, not a formal security audit or guarantee against loss.*
