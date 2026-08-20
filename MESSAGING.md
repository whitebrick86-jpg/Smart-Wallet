# Messaging

**Product:** Smart Wallet (Chrome / Opera MV3)  
**Audience:** Users, Chrome Web Store reviewers, operators  
**Live unpacked product:** 0.11.388  
**Consent notice:** 2026-08-20 (`smart_wallet_messaging_consent_v1`)  
**Date:** 2026-08-20  

This page is the product map for **Inbox / Messaging**: every panel, folder, and button, and the difference between **Delete conversation**, **Delete for me**, and **Request deletion of my server messages**.

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

Announcements are **read-only notices** in the same Messaging panel. They are not personal mail.

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

**Refresh inbox** appears when Inbox is open. It signs a pull for wallets on this device. It does not run in the background on Home, Send, Swap, or Bridge.

---

## 5. Compose

| Control | What it does |
|---------|----------------|
| **From** | Wallet on this device that will send |
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
| Each row | Shortened address + **Unblock** (and Copy). No message body is loaded. |
| Empty | **You haven’t blocked any addresses.** |

**Unblock**

- Pending: **Unblocking sender…**
- Success after HTTP 200: **Sender unblocked.**
- The panel stays open. Only that row disappears.
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

**Included (customer):** Inbox, Sent, Compose, Announcements (read), Refresh, local delete conversation / selected / clear, Delete for me, Block, Blocked addresses, Report, Request deletion of my server messages, messaging consent, privacy copy.

**Omitted (store strip):** Broadcast announcements composer, Owner tools / Message reports, owner device enrollment, admin-panel token, Managed RPC owner switches.

Canonical unpacked Messaging talks to the **production** mail host. A separate `STAGING-NOT-FOR-STORE` copy is not a Store package.

---

## 13. Limits and honesty

- Body cap **4,000 UTF-8 bytes** (multi-byte characters count as bytes).  
- Personal mail rate limits exist on the relay (send, report, block).  
- Report review is owner-only and stripped from Store.  
- Blocking does not affect chain Send.  
- Production Worker mail-privacy (delete-for-me, block, report, delete-all) was proven on **staging**. Production Worker **0.2.12** was not updated in that pass. Server buttons must not be offered as working in the Store listing until production matches.

---

*Not financial advice. Do not send seed phrases or private keys through Messaging.*
