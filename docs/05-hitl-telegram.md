## Section 5 — Human-in-the-Loop (Telegram approval setup)

HITL means that for every step that moves money or creates a legal or tax document, n8n **pauses** and sends your phone a message. Nothing continues until you decide. A paused workflow costs nothing and waits up to 24 hours.

### Step 1 — Create the bot with BotFather (5 minutes)
1. Install **Telegram** on your phone and open it.
2. In the search bar type **@BotFather**. Open the one with the blue ✔ verified tick.
3. Tap **Start**, then send: `/newbot`
4. BotFather asks for a name. Send: `Mukwa AIOS Approvals`
5. BotFather asks for a username ending in `bot`. Send: `mukwa_aios_bot`. If it is taken, try `mukwa_aios_approvals_bot`.
6. BotFather replies with a **token** like `7412345678:AAH3x…`. **Copy it and keep it secret.** Anyone with this token can control the bot.
7. Optional hardening. Send `/setjoingroups` → choose your bot → `Disable`, so nobody can add the bot to random groups.

### Step 2 — Get your chat ID
1. In Telegram search **@userinfobot** → **Start**. It replies with `Id: 123456789`. That number is your chat ID.
2. Open **your** bot (search `@mukwa_aios_bot`) and tap **Start**. *This step is required: a bot cannot message you until you have messaged it first.*
3. **Backup approver** (Week 12): ask them to do steps 1–2 on their phone, and keep their ID too.
4. *(Optional, recommended)* For a shared approvals group: create a Telegram group "Mukwa Approvals" → add the bot and the backup approver. Then send any message in the group, open `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates` in a browser, and copy the `chat":{"id":-100…` number. Group IDs start with `-100`. Use that ID instead.

### Step 3 — Configure the n8n Telegram credential
1. n8n → **Overview → Credentials → Create Credential** → search **Telegram API** → Continue.
2. **Access Token:** paste the BotFather token → **Save**. A green "Connection tested successfully" message appears.
3. In each AIOS workflow, open the **Config** node → `telegramChatId` → replace `123456789` with your ID → Save.

### Step 4 — The "Human Review" node (what you will see in n8n)
In this build, the human-review step is n8n's **Telegram → Send and Wait for Response** operation (titled e.g. **Telegram – Request Approval**). Open it and you will see:
- **Resource:** Message · **Operation:** Send and Wait for Response
- **Chat ID:** `{{ $('Config').first().json.telegramChatId }}`
- **Message:** the approval template (Step 5)
- **Response Type:** *Custom Form*, with three fields:
  - **Decision**, a dropdown: Approve / Deny / Need More Info (required)
  - **Reviewer**, text: your name (required, and written to the audit log)
  - **Notes**, text area: the reason, required in practice for Deny or Need More Info
- **Options → Limit Wait Time:** 24 hours
- **Options → Message Button Label:** `✅ Approve / ❌ Deny / 📝 Need More Info`

On your phone you get one message with one button. Tapping it opens a small n8n web page with the three choices and a Submit button. This works on n8n Cloud out of the box. If you self-host, your n8n address must be reachable from the internet over https.

> **Why a form and not three Telegram buttons?** n8n's built-in approval mode offers only two buttons (Approve / Decline) and cannot record a reason. The form adds the third option and captures *who* decided and *why*, which you need for the audit trail (Section 7). If you prefer pure speed for low-risk approvals, set **Response Type → Approval → Approve and Disapprove**. That gives two one-tap buttons, and the output becomes `$json.data.approved` (true/false).

### Step 5 — Approval message template

Invoice approval as it arrives on your phone (from W2, real field names):

```
🧾 Invoice approval needed
Agent: Invoicing Agent
Order: ORD-2026-0147
Action: Create QuickBooks invoice, then issue fiscal invoice in ZRA Smart Invoice

Buyer: Chongwe Builders Ltd (TPIN 1003456789)
4.5 m³ Teak Furniture-grade KD @ ZMW 24,990.00/m³
Subtotal ZMW 112,455.00
VAT 16% ZMW 17,992.80
TOTAL ZMW 130,447.80

Risk: low

[ ✅ Approve / ❌ Deny / 📝 Need More Info ]
```

Supplier quote approval (from W4):

```
🌳 Supplier quote
Agent: Sourcing Agent
Supplier: Kabompo Timber Concessions
• Kiaat Select sawn: USD 610/m³, 25 m³, 21 days
Compared: USD 610 vs last price USD 580 (+5.2%). Cheapest Kiaat supplier on file.
Legality docs: yes
Recommendation: ACCEPT

Approve = pass to Procurement for a draft PO. No money moves yet.

[ ✅ Approve / ❌ Deny / 📝 Need More Info ]
```

### Step 6 — What happens on each decision

| Decision | Invoice workflow (W2) | Supplier quote workflow (W4) |
|---|---|---|
| ✅ **Approve** | QuickBooks is checked for an existing invoice with DocNumber = OrderID. If none exists, the invoice is created → Orders.Status = Invoiced, InvoiceID filled → Approvals row = Approved (reviewer, time) → Telegram tells you to issue the ZRA fiscal invoice → AuditLog `HumanApproved=TRUE`. | Approvals row `ApproveSupplierQuote` / Approved is created; the Procurement Agent picks it up → AuditLog. |
| ❌ **Deny** | Approvals row = Denied with your reason → Order goes back to **Confirmed** (so it isn't picked up again) → AuditLog. Fix the order and set it to ReadyToInvoice again when ready. | Approvals row Denied with the reason → AuditLog. Nothing is sent to the supplier. |
| 📝 **Need More Info** | Approvals row = NeedsInfo with your question → Telegram confirms the order is parked → AuditLog. Edit the order: any edit while it is ReadyToInvoice re-triggers the workflow. | A **Gmail draft** to the supplier with the agent's clarifying questions plus your note → AuditLog. You review and send. |
| ⏱ **No answer in 24 h** | The workflow resumes with an empty decision, which counts as **not approved**, so the Need-More-Info path runs and the item stays parked. The Friday report lists it. | The Deny path runs with the reason "No response within 24 h". |

**Rule:** an empty answer must never count as approval. All routing checks for the exact word `Approve`.
