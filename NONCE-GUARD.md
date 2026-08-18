# Nonce Guard

**Product:** Smart Wallet (Chrome / Opera MV3 extension)  
**Applies to:** Ethereum and the other supported EVM networks  
**Does not apply to:** Solana, Bitcoin, or Sui  
**Repository:** Documentation only — extension source is **not** published here.

This document explains **what a nonce is**, why EVM wallets must treat it carefully, and **what Smart Wallet’s Nonce Guard does** when you send, swap, bridge, approve a token, or confirm a dApp transaction.

---

## 1. What a nonce is

On Ethereum-style networks (Ethereum, Polygon, Base, BNB Smart Chain, Arbitrum, Optimism, Avalanche C-Chain, Robinhood Chain, and similar), every account has a **nonce**.

A nonce is **not** a password, a secret, a fee, or a transaction hash. It is a **counter**: a whole number that starts at `0` and increases by one for each transaction that account has had **included in a block**.

Think of it as a ticket number at a counter:

| Account history | Next nonce the network will accept |
|-----------------|-------------------------------------|
| This address has never sent anything | `0` |
| One transaction has confirmed | `1` |
| Twenty-eight transactions have confirmed | `28` |

The network uses that number for two jobs at once:

1. **Order.** Transaction `28` cannot be mined before transaction `27` from the same sender is mined.
2. **Uniqueness.** You cannot replay an old signed transaction as a new one, because the old nonce is already used.

The word “nonce” in cryptography sometimes means “a random number used once.” That is **not** what EVM account nonces are. An EVM nonce is **sequential and public**. Anyone can look it up for any address. It does not hide your keys or your balance.

### 1.1 Confirmed nonce vs pending nonce

Nodes report two useful counts for an address:

| Count | Meaning |
|-------|---------|
| **Latest** (`eth_getTransactionCount(address, "latest")`) | How many of that address’s transactions are already in a **confirmed** block. This is the next nonce the chain will accept **after** everything currently mined. |
| **Pending** (`eth_getTransactionCount(address, "pending")`) | How many transactions the node has seen from that address, **including ones still sitting in the mempool** (broadcast but not mined yet). |

If latest and pending are the same, that account has **no unfinished transactions** on that node. The next new send can use nonce = latest.

If **pending is greater than latest**, something from that account is still waiting to mine. Example:

- Latest = `28`
- Pending = `31`

The node believes nonces `28`, `29`, and `30` are **already reserved** (in the mempool or otherwise counted as pending). The next *new* independent transaction would be nonce `31`. Smart Wallet treats that situation as a **queue**, not as a green light to send `31` by default.

### 1.2 Why a later nonce can strand funds

EVM nodes process one sender’s transactions **in nonce order**.

If nonce `28` is stuck (low fee, crowded mempool, or a dApp transaction that never mines), then:

- Nonce `29` waits behind `28`
- Nonce `30` waits behind `29`
- A brand-new send that takes nonce `31` also waits

The native token reserved for those later transactions (gas, and any value they move) can sit unused until `28` confirms **or** is replaced. That is how a wallet can look like it “has POL / ETH” while Send still says funds are reserved.

Replacing a stuck transaction does **not** use a new nonce. It reuses the **same** nonce with a higher fee. The network keeps at most one pending transaction per sender+nonce; the higher-paying replacement can displace the older one.

### 1.3 Two kinds of replacement

| Action | Same nonce? | What it does |
|--------|-------------|--------------|
| **Speed Up** | Yes | Re-sends the **same payload** (same recipient, amount, and calldata) with a higher fee so it can mine sooner. |
| **Cancel** | Yes | Sends a **new** transaction: to yourself, value `0`, empty data, **same nonce**. If that cancel mines first, the original pending transaction is discarded. You pay only network gas if the cancel wins. |

Neither action should invent nonce `29` while `28` is the earliest pending nonce.

### 1.4 Transaction types and fees (why replacement is picky)

EVM transactions come in a few types. The important ones for this guard:

| Type | Common name | Fee fields |
|------|-------------|------------|
| Type `0` | Legacy | `gasPrice` only |
| Type `1` | Access list (EIP-2930) | `gasPrice` only |
| Type `2` | EIP-1559 | `maxFeePerGas` and `maxPriorityFeePerGas` — **no** `gasPrice` |

Polygon, Ethereum, Base, and most Smart Wallet EVM chains use type `2` for wallet-originated transactions. BNB Smart Chain still uses legacy `gasPrice`.

To replace a pending transaction, the new fees must **beat** the original’s fees (and usually the current network recommendation). Nodes typically require a percentage bump (Smart Wallet uses integer wei arithmetic, rounds required values **up**, and adds a small safety margin). Polygon also has a minimum priority fee.

An empty RPC value such as `"0x"` is **not** a real fee. Smart Wallet must not treat it as zero and must not guess the transaction type from it.

A **transaction hash** on Ethereum/Polygon is 32 bytes, written as `0x` plus **64 hexadecimal characters**. That is how History recognizes an accepted hash. It is not a 64-byte length check.

---

## 2. What the Nonce Guard is

The **Nonce Guard** is Smart Wallet’s rule that the wallet must not silently create a **new independent nonce** while an earlier nonce from the same account is still pending on that EVM chain.

It is shared by:

- Send
- Internal swap
- Internal bridge
- Token approve / revoke
- dApp `eth_sendTransaction`
- Ledger and software signers

It is **per account + per chain**. A pending Polygon nonce does not block an Ethereum send. A pending Ethereum nonce does not block Solana.

Solana, Bitcoin, and Sui **do not use EVM account nonces** and are **unchanged** by this system.

---

## 3. What the guard does, step by step

### 3.1 Before any new independent transaction is signed

Immediately before signing, Smart Wallet:

1. Lists the RPC hosts for that chain.
2. **Deduplicates** identical URLs so the same host is not counted twice.
3. On each pinned host, queries **`latest` first, then `pending`**, sequentially (not in parallel fan-out).
4. Builds a picture of the queued nonce range, if any.

Then:

| Finding | Result |
|---------|--------|
| Latest equals pending | Allow the new transaction. |
| Pending greater than latest | **Block** a new independent nonce by default. |
| Hosts disagree about the gap | **Do not sign.** Treat it as disagreement, not a guess. |
| Counts cannot be read | **Do not sign.** Show that nonce status is unavailable. |

The user is told the **chain name**, the **native token** (ETH, POL, BNB, AVAX, …), the **earliest pending nonce**, and the queued range when it is known.

### 3.2 What you can do when a nonce is pending

The guard offers three safe choices for the **earliest** pending nonce:

| Choice | Meaning |
|--------|---------|
| **Wait** | Close the warning. Do not sign. The earlier transaction may still confirm. This is a waiting state, not “Send failed.” |
| **Speed Up earliest** | Open the fee review for the earliest nonce, using the **original** recipient, amount, and data if they can be fully recovered. |
| **Cancel earliest** | Open the fee review for a **zero-value self-send** on that same earliest nonce. |

There is an Advanced option to queue a **later** nonce anyway. It is hidden behind a strong warning and a confirmation checkbox. The **default remains blocked**. Closing or rejecting the warning signs nothing and broadcasts nothing.

### 3.3 What is explicitly allowed

Cancel and Speed Up of the **earliest** pending nonce are **exempt** from the “no new nonce” rule, because they **reuse** that nonce. They are not a new independent transaction.

They still must:

- Use the same account and chain
- Use that earliest nonce, not the next one
- Pass fee and payload checks (see below)
- Take the account+chain signing reservation so two windows cannot sign at once

### 3.4 Reservation (one signer at a time)

While a transaction is being prepared or signed for an account on a chain, Smart Wallet holds a **background-owned reservation** for that pair. Another Send, swap, or dApp request for the same account+chain cannot sneak in a second signature.

- Ledger reservations do not expire on a short timer (you may be confirming on the Nano).
- Software reservations can be cleaned up if abandoned.
- The reservation is renewed with a heartbeat while the sign is live.
- Success or failure **releases** it.

### 3.5 Recheck before sign and before broadcast

The latest/pending sample is taken again immediately before signing and again before broadcast. If the picture has changed — pending cleared, earliest nonce moved, or providers now disagree — Smart Wallet **does not sign** (or does not broadcast a stale replacement). It asks you to refresh rather than submitting the wrong nonce.

The nonce that actually gets signed must match the nonce that was reviewed. Software signing uses the **signed** nonce from the parsed signed transaction, not an unsigned draft field.

---

## 4. Speed Up vs Cancel (the safety boundary)

These two actions look similar in the UI. They are **not** the same recovery problem.

### 4.1 Speed Up — original payload required

Speed Up must reconstruct the original transaction:

- Recipient (`to`)
- Value
- Data (calldata)
- Chain
- Sender
- Nonce
- Original fee fields of the correct type

If any of those are missing, Smart Wallet **refuses**. It does not invent `value = 0`. It does not turn a send into a cancel. It does not offer a “guess the fees” Speed Up.

### 4.2 Cancel — same nonce, new empty self-send

Cancel **intentionally** builds a new transaction:

- `to` = your own address
- `value` = `0`
- `data` = `0x` (empty)
- `nonce` = earliest pending nonce
- gas limit typically `21,000` for a plain EOA self-send
- type `2` (EIP-1559) on chains that support it; legacy `gasPrice` on BNB Smart Chain
- no token approval, swap, bridge, contract call, or platform fee

If the original fees **can** be verified, Cancel uses the normal replacement policy: bump every applicable original fee field, never go below the current network recommendation, honor Polygon’s minimum tip, keep Max Fee ≥ Priority Fee and Max Fee ≥ base fee + tip, and show Recommended / Faster only after they pass the same validator used at sign time.

### 4.3 When original fees cannot be verified

Sometimes Smart Wallet knows a nonce is pending but **cannot recover** the original fee fields (no accepted local hash, no usable stored metadata, RPC `eth_getTransactionByHash` has nothing to fetch).

In that case:

- **Speed Up stays blocked.**
- **Cancel** may still proceed as a **manual** cancel using **current network fee estimates**.

This path is labeled **Manual Cancel — original fee unknown**. Recommended and Faster fees on that path are **current-network estimates, not guaranteed replacement minimums**.

#### Disclaimer — Manual Cancel when the original fee is unknown

Read this before using Manual Cancel. It is documented here instead of the popup so the fee review and Advanced fields stay usable.

- The original transaction’s fee **cannot be recovered**.
- Smart Wallet **cannot calculate a guaranteed replacement minimum**.
- The cancellation **may be rejected as underpriced**.
- The original transaction **may confirm first**.
- **Only network gas** is spent if the cancellation wins. Smart Wallet does not charge a platform fee for Cancel.
- The operation uses the **earliest pending nonce** on the current EVM chain (for example Polygon).
- It will **not** Speed Up or preserve the original transaction.
- It will **not** create the next nonce.
- Fees shown are **current-network estimates**, not guaranteed replacement minimums.
- If the network returns underpriced: nothing was accepted, History does not get a normal pending row, and you can raise the fee and retry the **same** nonce.

Speed Up never uses this fallback.

Smart Wallet first tries, in order:

1. An accepted transaction hash in local History (`0x` + 64 hex characters)
2. Same-nonce replacement-group records that already have accepted hashes
3. Stored non-cancel signed metadata that is structurally verifiable
4. `eth_getTransactionByHash` on deduplicated pinned RPC hosts
5. The existing allowlisted provider `getTransaction`

It will not treat a hashless “Replacement not submitted” row, a failed local attempt, or a cancel-shaped leftover as the original.

---

## 5. Replacement fees (what you review before Ledger or software sign)

The fee review shows, when known:

- Original transaction type
- Original Gas Price (legacy) or original Max Fee and Priority Fee (type 2)
- Minimum acceptable replacement values (when the original is verified)
- Recommended and Faster
- Estimated maximum network cost in the native token (for example POL)
- The **same nonce** being replaced

Units are labeled **Gwei** or the native coin.

Advanced inputs are **Gwei**, parsed with integer `parseUnits(..., 9)`. Blank required fields are errors. Empty `"0x"` is omitted, not sent as a number. The wallet does not silently overwrite the Advanced values you typed. If they are too low, they stay on screen with the exact required minimum.

Reviewed fees are what get signed. The Ledger / software path must **not** rebuild them with a later gas clamp or congestion bump. Type 2 replacements never include `gasPrice`. Legacy replacements never receive type-2 fields.

If the network says the replacement is underpriced:

- Nothing is treated as submitted
- History does not get a normal pending row
- You can raise the fee and retry the **same** nonce
- Smart Wallet does not automatically sign again

---

## 6. History

A failed validation, rejected signature, signing error, or RPC rejection **without** an accepted hash must not appear as an ordinary sent transaction (including a fake “-0”).

When a cancel **is** accepted by the network:

- Label: **Cancel pending transaction**
- Hash, nonce, status, gas, and self-destination are stored
- It is **not** counted as an outgoing transfer amount
- It is **not** shown as “Sent -0”

A Speed Up keeps the original transaction’s semantic label and amount.

Multiple hashes that share one nonce are **replacements of each other**, not separate outgoing spends.

---

## 7. What the Nonce Guard never does

- Invent a nonce
- Silently increment to the next nonce because one is busy
- Auto Speed Up or auto Cancel
- Repeatedly prompt Ledger after a failure
- Permanently poll the chain in the background
- Share one reservation across two accounts or two chains
- Apply this logic to Solana, Bitcoin, or Sui
- Charge a Smart Wallet platform fee for Send, Cancel, or Speed Up  
  (internal swap remains **45 bps** / 0.45%; internal bridge remains **85 bps** / 0.85%)

dApp requests that are blocked get a standard EIP-1193 error (`-32002` when blocked, `4001` when you reject). Closing the warning is a reject: no signature, no broadcast.

---

## 8. Supported EVM chains

The same guard and replacement rules are shared by all eight Smart Wallet EVM networks:

| Network | Native token |
|---------|----------------|
| Ethereum | ETH |
| Optimism | ETH |
| BNB Smart Chain | BNB (legacy gas price) |
| Polygon | POL |
| Robinhood Chain | ETH |
| Base | ETH |
| Arbitrum One | ETH |
| Avalanche C-Chain | AVAX |

---

## 9. Short picture

```text
You click Send / Swap / Bridge / Approve / dApp Confirm
        │
        ▼
  Sequential RPC: latest, then pending  (deduped hosts)
        │
        ├─ no pending gap ──────────► sign the new transaction
        │
        ├─ hosts disagree / no data ► do not sign
        │
        └─ pending > latest
              │
              ▼
         Show earliest nonce + queued range
              │
              ├─ Wait          ► no signature
              ├─ Speed Up      ► same nonce, original payload required
              ├─ Cancel        ► same nonce, zero self-send
              └─ Advanced queue ► only after explicit warning + checkbox
```

The Nonce Guard exists so a stuck nonce `28` cannot quietly become a pile of later nonces that reserve your ETH or POL until someone notices. You either wait, replace the earliest nonce, or — only if you mean to — queue behind it on purpose.

---

## Related docs

- [ARCHITECTURE.md](./ARCHITECTURE.md) — how signing and RPC are layered  
- [ERROR-SYSTEM.md](./ERROR-SYSTEM.md) — how “pending reserved,” underpriced, and nonce errors are named  
- [CHAINS.md](./CHAINS.md) — supported networks  
- [INTERNAL-DEX.md](./INTERNAL-DEX.md) — swap confirm truth (approval must confirm before a dependent swap signs)  
- [MODULES.md](./MODULES.md) — runtime module inventory  
