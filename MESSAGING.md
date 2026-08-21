# Messaging

**Product:** Smart Wallet (Chrome / Opera MV3)  
**Audience:** Users, Chrome Web Store reviewers, operators  
**Documentation target:** Ledger Messaging authorization release (set the exact wallet version before publishing)  
**Consent notice:** 2026-08-20 (`smart_wallet_messaging_consent_v1`)  
**Updated:** 2026-08-21  

This page is the product map for **Inbox / Messaging**: every panel, folder, and button, and the difference between **Delete conversation**, **Delete for me**, and **Request deletion of my server messages**.

Messaging supports software wallets and explicitly authorized Ledger accounts. A Ledger account must complete a one-time on-device authorization before it can send, reply, refresh personal mail, or manage its messaging data. The Ledger private key remains on the hardware device and is never exported to Smart Wallet.

Related: [PRIVACY-POLICY.md](./Chrome-extension-store-for-reviewers/PRIVACY-POLICY.md) · [CHROME-WEB-STORE-READINESS.md](./CHROME-WEB-STORE-READINESS.md) · [DOCUMENTATION.txt](./DOCUMENTATION.txt)

---

## 1. What Messaging is

Messaging is **optional wallet-to-wallet mail** inside Smart Wallet. It is not a blockchain transfer, not WalletConnect chat, and not end-to-end encryption.

- You write to another **wallet address** you choose (saved contact or pasted address).
- Mail uses the Smart Wallet mail relay, not the chain.
- **“Sent”** means the relay **confirmed storage**. It does **not** mean the other person opened the message.
- Contents, public addresses, timestamps, IP address, and security metadata can be stored on Smart Wallet infrastructure for delivery, abuse prevention, and legal compliance.
- Messages are encrypted **in transit and at rest**. Authorized infrastructure can process contents. They are **not** end-to-end encrypted.
- Personal Messaging stays **inside the wallet window**. Buttons do not open an external website, tab, or browser window.
- Blockchain **Send / Swap / Bridge** is separate and is not blocked by a Messaging block list.
- Inbox and Sent show only the selected wallet identity’s conversations with each recipient. Switching wallets does not list another wallet’s threads.

Announcements are **read-only notices** in the same Messaging panel. They are not personal mail.

### 1.1 Ledger Messaging

Ledger Messaging lets a supported EVM or Solana Ledger address authorize Smart Wallet’s separate, non-exportable messaging key. The Ledger signs an off-chain authorization; it does not reveal its private key and does not sign every ordinary message.

The stable messaging identity is the selected Ledger address. The authorized messaging key may later be rotated without creating a different visible sender or a new conversation. Smart Wallet must resolve the recipient’s current authorized key before sending a new message instead of reusing a revoked key stored in an older message.

Ledger Messaging authorization is separate from blockchain transaction signing:

- It does not move tokens or authorize Send, Swap, or Bridge.
- It does not expose, copy, or store the Ledger private key.
- It does not allow a software-wallet key or infrastructure key to impersonate a Ledger address.
- It does not require the Ledger to remain connected for every ordinary message after authorization succeeds.
- EVM and Solana support are enabled separately and only after physical-device verification for that Ledger app and signing route.

#### Enable Ledger

**Enable Ledger** is the single authorization control. It handles both first-time authorization and authorization again after revocation, expiration, reinstall, or messaging-key rotation.

1. Select the Ledger-backed account that will be used as the messaging identity.
2. Unlock the Ledger, connect it to the computer, and allow the browser’s device connection when prompted.
3. Open the Ethereum app for a supported EVM account or the Solana app for a supported Solana account.
4. Press **Enable Ledger**.
5. Smart Wallet prepares an authorization that binds the selected Ledger address to the current messaging public key, Smart Wallet application/domain, chain, nonce, audience, and expiration.
6. Review the request and approve it on the physical Ledger.
7. Smart Wallet verifies the returned signature against the exact selected Ledger address.
8. Smart Wallet registers the verified messaging key with the mail service.
9. The interface shows **Ledger Messaging Authorized** only after signature verification and registration both succeed.

Expected control states are **Enable Ledger**, **Connecting to Ledger…**, **Approve on Ledger…**, and **Ledger Messaging Authorized**. Rejecting the request, disconnecting the Ledger, opening the wrong app, returning the wrong address, or failing backend registration leaves Messaging unauthorized. Smart Wallet never silently falls back to another account’s software key.

#### Revoke

**Revoke** invalidates all messaging authorization and active messaging sessions associated with the selected Ledger messaging identity in one action.

- Revocation stops revoked sessions from sending or receiving new personal messages as that Ledger identity.
- A new physical Ledger authorization is required before Ledger Messaging can be used again.
- Revocation does not remove the Ledger account, move funds, or affect Send, Swap, Bridge, or transaction signing.
- Revocation does not automatically delete conversation history or messages already delivered to another participant.
- If old history depends on an old encryption key, Smart Wallet must describe whether that history remains readable; revocation must not silently present itself as message deletion.

There is no separate **Reauthorize** button. After revocation or expiration, the same **Enable Ledger** control performs a fresh authorization. There is no **Revoke All** control when only one Ledger messaging identity is active; **Revoke** invalidates all sessions for the selected Ledger identity.

---

## 2. Before anything personal is sent or pulled

The first time you use personal Messaging (send, reply, refresh inbox, delete for me, block, report, or request server deletion), Smart Wallet shows a **local privacy and encryption notice** (version **2026-08-20**).

- Consent is stored **only on this device**. It is not sent to the Worker.
- It is **separate** from the Managed RPC / privacy-consent notice.
- If you decline, personal send/pull/delete-for-me/block/report/server-deletion does not run.
- Announcements can still be read.

Never put a seed phrase, private key, password, or recovery words in a message.

---

## 3. How to open it

**Settings → Inbox → Open**

Settings labels this block **Inbox**. The panel title is **Messaging**.

Settings also shows:

- Unread status
- Blocked senders (summary)
- **Request deletion of my server messages** (described below)

The Messaging header has **← Settings** and a vertical **⋮** (**Messaging options**) at the upper-right of the header.

---

## 4. Folders

| Folder | What it is |
|--------|------------|
| **Inbox** | Personal messages to a wallet on this device |
| **Sent** | Personal messages you sent, after the relay confirmed storage |
| **Compose** | New message form |
| **Announcements** | Product notices. Not a personal conversation. Store builds can **read** these; they cannot **broadcast** new ones |

**Refresh inbox** appears when Inbox is open. It authenticates the pull for the selected wallet identity. A Ledger account must already have a valid Ledger-authorized messaging key; Refresh does not request a new on-device signature every time. It does not run in the background on Home, Send, Swap, or Bridge. Empty Inbox: **No conversations for this wallet yet.**

---

## 5. Compose

| Control | What it does |
|---------|----------------|
| **From** | Selected software wallet or Ledger account with a valid Ledger Messaging authorization |
| **To** | Saved contact |
| **Or paste another wallet address** | Address that is not in contacts. From and To cannot be the same |
| **Subject** | Optional; capped |
| **Message** | Body; maximum **4,000 UTF-8 bytes** |
| **Send** | After consent, posts to the mail relay. Pending is honest. Success becomes **Sent** only after confirmed storage. Failure stays **Not delivered**. No retry loop. |

Reply in an open conversation uses the same storage rule: **Reply** does not toast success until storage is confirmed.

Delivery labels on outgoing bubbles: **Sending…**, **Sent**, **Not delivered**. Color is not the only signal.

---

## 6. Inbox tools (this device)

These tools change **this device’s copy**. They do **not** delete the other person’s relay copy.

Open **Delete Messages** to reveal the extra chips. **✕** exits that mode.

| Button | What it does | Server? |
|--------|----------------|---------|
| **Delete Messages** | Shows select / delete / clear tools | No |
| **✕** (exit delete messages) | Hides those tools | No |
| **Select all** | Selects visible Inbox conversations | No |
| **Delete selected** | Removes selected conversations from this device and remembers them so a later pull does not restore them | **No** |
| **Clear inbox** | Asks “Clear all Inbox conversations on this device?” then removes Inbox threads locally | **No** |

Toasts: `Deleted N.` / `Inbox cleared.` / `Inbox already empty.` / `Select a message first.`

---

## 7. Open conversation

| Button | Where | What it does |
|--------|--------|----------------|
| **Back to list** | Top row, left | Closes the conversation and returns to the folder list. Does not delete. |
| **Delete conversation** | Top row, right | **Local only.** Removes that conversation from this device. Toast: **Conversation deleted.** The other participant may still have their copy. This is **not** Delete for me and **not** a server deletion request. |
| **⋮ Message actions** | Upper-right of the conversation meta row (near the message count) | Opens a menu: Delete for me, Block sender, Report message |

### 7.1 Delete for me

Menu: **Delete for me**

1. You confirm: “Delete this message for you?” The copy explains the other participant may retain their copy.  
2. Pending toast: **Deleting message…**  
3. The wallet you control signs `sw-mail-delete-v1` for that message id.  
4. The relay removes **your** Inbox copy.  
5. Success toast (only after HTTP success): **Message deleted for you.**  
6. This device also drops the local conversation row.  
7. Failure: **Could not delete the message. Try again.** The message stays visible.  
8. Double-click starts **one** operation.  
9. Cancel / Back does not delete.

This is **not** a claim that the message is gone everywhere.

### 7.2 Block sender

Menu: **Block sender**

1. Confirm: future personal messages from that address will not be stored for **your** Inbox. Token sends are not affected. Existing messages stay until you delete them.  
2. Pending: **Blocking sender…**  
3. Success after HTTP 200: **Sender blocked.**  
4. Failure: **Could not block this sender. Try again.**  
5. The other party is **not** told they are blocked. A later send from them fails with a generic rejection.  
6. You stay in Messaging. No external page opens.

### 7.3 Report message

Menu: **Report message**

1. You explicitly confirm consent to review **that selected message** (not unrelated conversations).  
2. Pending: **Submitting report…**  
3. Success after confirmed storage: **Message report submitted.** That does **not** mean enforcement happened.  
4. Failure: **Could not submit the report. Try again.** No local “already submitted” fake state.  
5. The HTTP response, toast, and logs must not show the body.  
6. Only a participant can report.

---

## 8. Blocked addresses (header ⋮)

Header **⋮** → **Blocked addresses** (menu first, then panel).

| Control | What it does |
|---------|----------------|
| **⋮** / **Messaging options** | Inside the Messaging header, upper-right below the address bar. About 40×40 CSS pixels. Enter / Space activate it because it is a real button. `aria-label="Messaging options"`. |
| **Blocked addresses** (menu item) | Opens the **internal** panel. Does not navigate away. The list request runs when the panel opens, not before. |
| Panel title | **Blocked addresses** |
| **✕** | Closes the panel. There is no “← Messaging” back arrow; **✕** is the exit. |
| **Select all** | Selects every listed blocked address |
| **Unblock selected address** | Unblocks the checked rows. There is no Clear all. Rows do not repeat Copy/Unblock. |
| Each row | Checkbox + shortened address. No message body is loaded. |
| Empty | **You haven’t blocked any addresses.** |

**Unblock selected**

- Pending: **Unblocking sender…** (or **Unblocking selected addresses…**)
- Success after HTTP 200: **Sender unblocked.** (or **Selected addresses unblocked.**)
- The panel stays open. Unblocked rows disappear.
- After unblock, that address may send a new personal message.

**Load failure:** **Could not load blocked addresses. Try again.**  
**Unblock failure:** **Could not unblock this sender. Try again.**

Block records have **no message bodies**. New records have **no 365-day TTL**; they persist until Unblock or a verified deletion. Another wallet cannot list or change yours. Server cap is **200** addresses.

Lock, unlock, and reload should keep the server block list. Blockchain Send remains available.

---

## 9. Request deletion of my server messages

**Settings → Inbox → Request deletion of my server messages**

This is the **full messaging-data deletion request**. It is not Delete conversation and not Delete for me.

1. Consent notice if not already accepted.  
2. Confirm: this requests deletion of **eligible Smart Wallet Inbox copies for a wallet you control**. Other participants’ copies, screenshots, or legally retained records may remain.  
3. Pending: **Submitting deletion request…**  
4. Success: **Messaging-data deletion request submitted.** The UI does **not** claim deletion is complete.  
5. The request identifier may appear only in the private Settings status line (`settingsMailPrivacyStatus`), not in the toast.  
6. Failure: **Could not submit the deletion request. Try again.**  
7. Signed domain: `sw-mail-delete-all-v1`.  
8. No message body is placed in the toast or logs.

---

## 10. The three deletes, compared

| Action | Where | What is removed | Other person | Honest success copy |
|--------|--------|-----------------|--------------|---------------------|
| **Delete conversation** | Open thread, red chip | This device’s conversation row | Keeps their copy | **Conversation deleted.** |
| **Delete selected / Clear inbox** | Inbox tools | This device’s selected / all Inbox threads | Keeps their copies | **Deleted N.** / **Inbox cleared.** |
| **Delete for me** | Thread ⋮ menu | Your relay Inbox copy **and** the local row | May keep their relay copy | **Message deleted for you.** |
| **Request deletion of my server messages** | Settings → Inbox | Eligible **server** copies for a wallet you prove you control | Other copies may remain | **Messaging-data deletion request submitted.** (not “deleted”) |

If you only want it off this browser, use **Delete conversation** or Inbox tools.  
If you want your relay Inbox copy gone, use **Delete for me**.  
If you want a recorded request to delete eligible server messaging data for your wallet, use **Request deletion of my server messages**.

---

## 11. Bottom notifications

Pending, success, and failure use the existing bottom toast on the current Smart Wallet surface:

- Green success style, neutral pending style, existing error style  
- Text is not conveyed by color alone (`aria-live`)  
- No message bodies, complete addresses, tokens, or signatures in the toast  
- Success only after authenticated HTTP 200 where the action is a server call  
- Pending buttons are disabled; double click is one request  
- Closing the popup does not invent success

| Action | Pending | Success | Failure |
|--------|---------|---------|---------|
| Delete for me | Deleting message… | Message deleted for you. | Could not delete the message. Try again. |
| Deletion request | Submitting deletion request… | Messaging-data deletion request submitted. | Could not submit the deletion request. Try again. |
| Block | Blocking sender… | Sender blocked. | Could not block this sender. Try again. |
| Load blocked list | Loading blocked addresses… | (list or empty state) | Could not load blocked addresses. Try again. |
| Unblock | Unblocking sender… | Sender unblocked. | Could not unblock this sender. Try again. |
| Report | Submitting report… | Message report submitted. | Could not submit the report. Try again. |

Owner-only (unpacked, not Store): Loading reported message… / Updating report… / Resolving report… / Dismissing report… with matching success and failure lines.

---

## 12. What Store builds include vs omit

**Included (customer):** Inbox, Sent, Compose, Announcements (read), Refresh, local delete conversation / selected / clear, Delete for me, Block, Blocked addresses, Report, Request deletion of my server messages, messaging consent, privacy copy, and Ledger Messaging authorization/revocation when the applicable chain feature has passed physical-device validation and is enabled for that build.

**Omitted (store strip):** Broadcast announcements composer, Owner tools / Message reports, owner device enrollment, admin-panel token, Managed RPC owner switches.

Canonical unpacked Messaging talks to the **production** mail host. A separate `STAGING-NOT-FOR-STORE` copy is not a Store package.

---

## 13. Limits and honesty

- Body cap **4,000 UTF-8 bytes** (multi-byte characters count as bytes).  
- Personal mail rate limits exist on the relay (send, report, block).  
- Report review is owner-only and stripped from Store.  
- Blocking does not affect chain Send.  
- Production Worker mail-privacy (delete-for-me, block, report, delete-all) is live on production **0.2.20**. Before publishing this document, verify that the deployed mail service accepts and verifies the Ledger Messaging authorization protocol described above.

---

## 14. Ledger Messaging FAQ

### Does Enable Ledger import my Ledger private key?

No. The private key remains inside the Ledger. The device signs an off-chain authorization linking the selected public Ledger address to Smart Wallet’s non-exportable messaging public key.

### Why must I approve something on the physical Ledger?

The messaging key is separate from the cryptocurrency wallet key. Physical approval proves that the selected Ledger address authorized that particular messaging key. A device key or another software wallet key cannot provide that proof.

### Do I approve every message on the Ledger?

No. The Ledger authorizes the messaging key. After that succeeds, the authorized messaging key handles ordinary messages until the authorization expires or is revoked.

### What happens if I press Enable Ledger after previously authorizing?

If the authorization is still valid, Smart Wallet shows the authorized state. If it was revoked, expired, lost during reinstall, or replaced during key rotation, **Enable Ledger** starts a fresh connection and physical authorization. A separate Reauthorize button is unnecessary.

### What does Revoke do?

Revoke invalidates all messaging authorization and active sessions for the selected Ledger identity. It does not remove the Ledger account or affect funds and blockchain activity. Messaging cannot resume for that identity until **Enable Ledger** completes a new physical authorization.

### Does Revoke delete my messages?

No. Revocation ends authorization and sessions; it is not a deletion command. Use the separate conversation and messaging-data deletion controls described in this document. Messages already delivered to another participant may remain in that participant’s copy.

### Can I continue in the same conversation after revoking and enabling again?

Yes. The thread is tied to the stable wallet addresses, not permanently to the old messaging key. After successful authorization, the same Reply field becomes available again. Before sending, Smart Wallet resolves the recipient’s current authorized key and uses the sender’s current authorized key. It must not send to a revoked key copied from an older message.

### Does a new messaging key create a different sender?

No. The visible identity remains the same Ledger address. Individual messages may internally reference different authorized key versions, but key rotation does not create a new visible wallet identity or conversation.

### Can I type while authorization is revoked?

The wallet may preserve an unsent draft locally, but it must disable sending and clearly show that Ledger Messaging authorization is required. It must not queue or claim delivery under a revoked identity.

### Why is there only one Revoke button?

One press revokes all sessions controlled by the selected Ledger messaging identity. Users should not need to revoke sessions individually. If Smart Wallet later supports several simultaneously authorized Ledger identities, each identity should still have its own Revoke action.

### Is Revoke the same as Request deletion of my server messages?

No. Revoke stops authorization and active sessions. **Request deletion of my server messages** submits a separate request concerning eligible stored messaging data. Neither action guarantees deletion of copies already held by another participant.

### What if the browser never asks to connect or the Ledger never shows an approval?

Authorization has not completed. Confirm that the Ledger is unlocked, connected, and running the correct Ethereum or Solana app; then try **Enable Ledger** again. Smart Wallet must not display **Ledger Messaging Authorized** unless it received and verified a physical-device signature and the mail service accepted the authorization.

### Can another Ledger device control the same messaging identity?

Only a device capable of producing a valid signature for the same Ledger-derived address can authorize that identity—for example, a replacement Ledger restored from the same recovery phrase. A Ledger with a different seed controls different addresses and cannot authorize the original identity.

### Does Ledger Messaging change normal Ledger wallet features?

No. Ledger Messaging authorization is separate from Ledger transaction signing. Enabling or revoking Messaging must not change Send, Swap, Bridge, account visibility, balances, or funds.

---

*Not financial advice. Do not send seed phrases or private keys through Messaging.*
