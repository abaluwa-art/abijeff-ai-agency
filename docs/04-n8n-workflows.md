## Section 4 — n8n Workflows (importable JSON)

The workflow files are in the `n8n/` folder of this package:

| File | Workflow name | Nodes | Trigger |
|---|---|---|---|
| `n8n/W0-AIOS-Error-Handler.json` | AIOS – Error Handler | 6 | Error Trigger (any AIOS workflow fails) |
| `n8n/W1-AIOS-Sales-Lead-Intake.json` | AIOS – Sales Lead Intake | 24 | Gmail label `Inquiries`, every minute |
| `n8n/W2-AIOS-Invoice-Preparation.json` | AIOS – Invoice Preparation | 39 | Airtable Orders, Status = ReadyToInvoice, every 15 min |
| `n8n/W3-AIOS-Inventory-Watchdog.json` | AIOS – Inventory Watchdog | 24 | Daily 07:00 Africa/Lusaka |
| `n8n/W4-AIOS-Sourcing-Email-Parser.json` | AIOS – Sourcing Email Parser | 28 | Gmail label `Suppliers`, every 10 min |
| `n8n/W5-AIOS-Sync-Orders-Leads-to-Sheets.json` | AIOS – Sync Orders & Leads to Sheets | 5 | Hourly. Copies Airtable to Sheets for the Dashboard (Section 10) |

Every file has been checked against the official n8n node definitions (n8n-nodes-base 2.15, @n8n/n8n-nodes-langchain 2.48). Each node type, version, parameter and option value exists in n8n. The two Code nodes were run against sample orders and stock rows. They produced the totals shown in Section 2. Each workflow is **inactive** on import, so nothing runs until you switch it on.

### 4.1 Import (repeat for each file, W0 first)

1. Open the `.json` file in any text editor (Notepad, TextEdit, or GitHub's **Raw** view) → **Select All** → **Copy**.
2. In n8n: **Overview → Create Workflow** (top right).
3. Click anywhere on the empty canvas → **Ctrl+V** (Mac: **⌘+V**). The nodes appear. Alternatively use **⋯ (top right) → Import from File…** and pick the `.json` file.
4. Click **Save** (top right). The workflow name comes from the file.
5. Nodes with a red ⚠️ need a credential or a dropdown selection. That is expected. Section 4.3 tells you what to pick.

### 4.2 Create credentials once (n8n → **Overview → Credentials** tab → **Create Credential**)

| Credential | Search for | What to do |
|---|---|---|
| Gmail | `Gmail OAuth2 API` | On n8n Cloud: **Sign in with Google** → choose the business Gmail account → Allow. Self-hosted: follow the n8n "Google OAuth" guide linked in the credential panel. |
| Google Sheets | `Google Sheets OAuth2 API` | Same Google account → **Sign in with Google** → Allow. |
| Airtable | `Airtable Personal Access Token API` | airtable.com → **Builder Hub → Personal access tokens → Create token**. Scopes: `data.records:read`, `data.records:write`, `schema.bases:read`. Access: base **AIOS_OPS** only. Paste the token. |
| OpenAI | `OpenAI API` | platform.openai.com → **API keys → Create new secret key** → paste. Then go to **Settings → Limits** and set a **monthly budget of USD 25**, a hard stop that protects you from runaway costs. |
| DeepSeek | `DeepSeek API` | platform.deepseek.com → **API keys → Create** → paste. Top up USD 5; that lasts months. |
| Telegram | `Telegram API` | Paste the bot token from BotFather (Section 5, step 1). |
| QuickBooks | `QuickBooks Online OAuth2 API` | Needs a free Intuit developer app; see 4.4. Environment: **Production** (use **Sandbox** while testing). |

### 4.3 Per-workflow setup (open each workflow and click each ⚠️ node)

**All workflows:**
1. Open the **Config** node and replace `telegramChatId` `123456789` with your chat ID (Section 5, step 2).
2. Go to **⋯ → Settings → Error Workflow** → select **AIOS – Error Handler** → Save. (W0 itself doesn't need this.)
3. Check that **Settings → Timezone** shows `Africa/Lusaka`. It is preset.
4. In every **Telegram** node, pick the Telegram credential.
5. In every **Airtable** node, pick the credential → **Base: AIOS_OPS** → **Table:** the table named in the node's title.
6. In every **Google Sheets** node, pick the credential → **Document: AIOS Sheets** → **Sheet:** `AuditLog`, `TokenCosts` or `Inventory`, matching the node's title.

**W1 – Sales Lead Intake**
1. In Gmail, create the label **Inquiries**, then add a filter: **Gmail → Settings → Filters → Create new filter** → *To:* `sales@yourdomain.co.zm` (or *Has the words:* `timber OR quote OR teak OR kiaat OR mahogany OR rosewood OR m3`) → **Apply the label: Inquiries**.
2. **Gmail Trigger – Inquiries** → pick the Gmail credential. The search `label:Inquiries` is preset.
3. **OpenAI gpt-4o-mini** → pick the OpenAI credential.
4. **Inventory Lookup (read-only)** → Sheets credential → AIOS Sheets → `Inventory`.
5. **Create Gmail Draft Reply** → Gmail credential.
6. Click **Test workflow**, then send a test email from your phone with the label applied (use email #1 from Section 9). Watch the nodes turn green, then look in Gmail → Drafts.

Flow: Gmail → Config → Prepare Email → **Find Existing Leads** (one lookup for the whole poll) → **Mark Already-Processed Emails** (Merge) → Lead Already Exists? → **Sales Agent** (+ model, memory, inventory tool, JSON schema) → Is Spam? → Create Lead → Needs Owner? → (escalate: Telegram ⚠️) / (normal: Gmail draft + Telegram FYI) → AuditLog + TokenCosts.

**W2 – Invoice Preparation**
1. **Airtable Trigger – ReadyToInvoice** → Airtable credential. In **Base** and **Table**, paste the URL of the Orders table from your browser (it looks like `https://airtable.com/app…/tbl…/viw…`). The trigger field `UpdatedAt` and the formula `{Status}='ReadyToInvoice'` are preset.
2. **Config** → fill in the QuickBooks values (see 4.4): `qboRealmId`, `qboVatTaxCodeId`, `qboItemMap`. While testing, set `qboBaseUrl` to `https://sandbox-quickbooks.api.intuit.com`.
3. Both **QBO –** HTTP nodes → *Credential for QuickBooks Online OAuth2 API* → pick your QuickBooks credential.
4. **OpenAI gpt-4o-mini** → OpenAI credential.

Flow: Airtable Trigger → Config → **Already Invoiced?** (idempotency 1) → **Pre-flight Checks & Maths** (Code: TPIN, delivery, species, QuickBooks customer, FX, VAT 16%, totals) → Pre-flight Passed? → **Invoicing Agent** review → Agent Says Valid? → Create Approval (Pending) → **Telegram – Request Approval** (waits up to 24 h) → Approved? → **QBO – Find Invoice by DocNumber** (idempotency 2) → Invoice Exists? → **QBO – Create Invoice** → Order → Invoiced → Approval → Approved → **Telegram: "Issue the fiscal invoice in ZRA Smart Invoice now"** → AuditLog. Denied → Approval Denied + Order back to Confirmed. Need More Info → Approval NeedsInfo + Telegram note. Any failed check → **dead-letter** (Approvals Status=Error) + Telegram ⛔.

**W3 – Inventory Watchdog**
1. **Read Inventory Sheet** → Sheets credential → AIOS Sheets → `Inventory`.
2. Pick credentials in OpenAI, Airtable and Telegram.
3. Click **Test workflow** once. With the sample rows you should receive a briefing about ROS-KD-50x150 (LOW) and KIA-AD-75x225 (STALE).
4. Toggle **Active** (top right). It now runs daily at 07:00 CAT.

Flow: Schedule → Config → Read Inventory → **Flag Low & Stale Stock** (Code) → Anything to Report? → (no: ✅ heartbeat) / (yes: **Inventory Agent** → Telegram briefing → Sourcing Task Needed? → split per alert → keep high/medium → **Find Open Sourcing Tasks** (one lookup) → **Keep SKUs Without Open Task** (Code) → Create Sourcing Task) → AuditLog.

**W4 – Sourcing Email Parser**
1. Create the Gmail label **Suppliers** and a filter that applies it to mail from your suppliers' addresses (e.g. *From:* `kabompotimber.co.zm OR mwinilungasawmills.co.zm OR zambeziteak.co.zm`).
2. **DeepSeek Chat (extraction)** → DeepSeek credential.
3. **Supplier Lookup (read-only)** and **Load All Suppliers** → Airtable credential → AIOS_OPS → Suppliers. Type every ContactEmail in Airtable in **lowercase**, because matching is exact.
4. Gmail, Airtable and Telegram credentials in the remaining nodes.

Flow: Gmail → Config → Prepare Email → **Sourcing Agent** (+ DeepSeek, memory, supplier lookup, JSON schema) → Is Supplier Quote? → Supplier Email Key → **Load All Suppliers** + **Match Supplier by Email** (Merge) → Supplier Exists? → Update price **or** Create supplier → **Telegram – Approve Supplier Quote?** → Approved: Approvals row `ApproveSupplierQuote` (for Procurement) / Need More Info: Gmail draft with questions to the supplier / Deny: Approvals row Denied → AuditLog.

**W0 – Error Handler**: pick the Airtable, Telegram and Sheets credentials → Save. It does not need to be Active. It runs whenever another workflow names it as its Error Workflow.

### 4.4 QuickBooks Online: credential and the IDs the Config node needs

**A. Create the Intuit app (one time, about 20 minutes)**
1. Go to **developer.intuit.com → Sign in** with your QuickBooks login → **Dashboard → Create an app → QuickBooks Online and Payments**. Name it `AIOS n8n` and choose scope **com.intuit.quickbooks.accounting**.
2. Open the **Keys & credentials** tab and copy the **Client ID** and **Client Secret** (Development keys for Sandbox; Production keys need Intuit's short questionnaire).
3. Under **Redirect URIs**, paste the **OAuth Redirect URL** shown in n8n's QuickBooks credential window (it looks like `https://yourname.app.n8n.cloud/rest/oauth2-credential/callback`).
4. In n8n's QuickBooks credential, paste the Client ID and Secret → Environment **Sandbox** first → **Connect my account** → choose the company.

**B. Find the IDs**
| Config field | Where to find it | Example |
|---|---|---|
| `qboRealmId` | QuickBooks → ⚙ → **Account and settings → Billing & subscription → Company ID**, without spaces | `9341453050123456` |
| `qboItemMap` | Create 4 products/services: **Sales → Products and services → New → Non-inventory**, named `Rosewood timber`, `Teak timber`, `Mahogany timber`, `Kiaat timber`, income account "Sales of timber". Open each one; the number at the end of the URL (`…item?itemId=22`) is its ID | `{"Rosewood":"21","Teak":"22","Mahogany":"23","Kiaat":"24"}` |
| `qboVatTaxCodeId` | Easiest method: in n8n, add a temporary HTTP Request node with the same QuickBooks credential, GET `https://quickbooks.api.intuit.com/v3/company/<realmId>/query?query=select * from TaxCode`, then find the code whose Name is your 16% standard-rate VAT and copy its `Id` | `5` |
| QBO_CustomerID (per order) | **Sales → Customers** → open customer → number at the end of the URL (`nameId=58`) | `58` |

Also turn on **Multicurrency** (⚙ → Account and settings → Advanced → Currency) if you invoice in both ZMW and USD. Each customer has a single currency, so a customer billed in USD must be a USD customer.

### 4.5 What is deliberately *not* automated

- **The QuickBooks invoice is not emailed to the customer.** The customer receives the **ZRA fiscal invoice**. Section 7 explains why.
- **Pricing is not chosen by AI.** You enter `unit_price_usd` in the order's LineItems.
- **Gmail replies are drafts.** You press Send.

### 4.6 Regenerating the workflows after a prompt change

The JSON files are built from `prompts/*.md` by `tools/build_workflows.py`. A non-technical owner can ignore this: just edit the prompt inside n8n and log the change in the Airtable **Prompts** table. A technical helper can edit the prompt file, run `python3 tools/build_workflows.py`, and re-import.
