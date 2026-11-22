## Section 6 — Idempotency & Error Handling

**Idempotency** means that running the same step twice has the same effect as running it once. Gmail can fire twice, n8n retries failed steps, and you might press "Test workflow" on a real order. None of these may produce a second invoice or a second PO.

### 6.1 The pattern: check first, then create

```
BEFORE every Create in QuickBooks / Airtable:
  1. Build a natural key you control         (OrderID, PO number, Gmail message ID, SKU+Status)
  2. Look the key up in the target system    (Search / Query node, "Always Output Data" ON)
  3. IF found  → do NOT create; link to the existing record; log "DuplicatePrevented"
     IF empty  → create, using the key as the record's number
  4. Write the new record's ID back to Airtable immediately
```

> **About "QuickBooks custom fields".** The QuickBooks API **cannot search by custom field**, so a query on an `order_id` custom field is not possible. This build therefore puts the OrderID into the invoice's **DocNumber** (invoice number), which *is* searchable. Order IDs must be ≤ 21 characters (e.g. `ORD-2026-0147`). If your ZRA integration controls the invoice number, keep DocNumber = OrderID in QuickBooks and record the ZRA number separately in `ZRA_InvoiceNo`.

### 6.2 Exact n8n configuration in W2 (invoice)

**Check 1 — node `Already Invoiced?` (IF, version 2.2)**

```
Conditions (combine with: OR)
  1. Value 1: {{ $json.fields.InvoiceID }}       Operation: String → is not empty
  2. Value 1: {{ $json.fields.ZRA_InvoiceNo }}   Operation: String → is not empty
true  → Audit – Already Invoiced (stop)
false → Pre-flight Checks & Maths
```

**Check 2 — node `QBO – Find Invoice by DocNumber` (HTTP Request)**

```
Method:          GET
URL:             {{ $('Config').first().json.qboBaseUrl }}/v3/company/{{ $('Config').first().json.qboRealmId }}/query
Authentication:  Predefined Credential Type → QuickBooks Online OAuth2 API
Query params:    query        = select Id, DocNumber, TotalAmt from Invoice where DocNumber = '{{ $('Pre-flight Checks & Maths').item.json.orderId }}'
                 minorversion = 75
Header:          Accept = application/json
Options:         Timeout = 30000 ms
Settings tab:    Retry On Fail = ON, Max Tries = 3, Wait Between Tries = 5000 ms
```

**Node `Invoice Exists in QBO?` (IF)**

```
Value 1: {{ ($json.QueryResponse?.Invoice || []).length }}   Operation: Number → is greater than   Value 2: 0
true  → Telegram – Duplicate Prevented → Order → Link Existing Invoice (no new invoice)
false → QBO – Create Invoice (DocNumber = OrderID)
```

**Same pattern elsewhere:**

| Workflow | Key | Lookup node | Setting |
|---|---|---|---|
| W1 Sales | Gmail message ID → `Leads.SourceMessageID` | `Find Existing Leads (idempotency)`: one Airtable Search for all emails in the poll, formula `OR({SourceMessageID}='id1',{SourceMessageID}='id2',…)` built automatically → **Merge** `Mark Already-Processed Emails` (Combine → by fields `messageId` = `SourceMessageID` → *Enrich Input 1*) → IF `{{ $json.id }}` exists | Search node Settings → **Always Output Data = ON** and **Execute Once = ON** |
| W3 Inventory | SKU + open status | `Find Open Sourcing Tasks (idempotency)`: `AND({Action}='SourcingTask', OR({Status}='Pending',{Status}='NeedsInfo'))` → Code `Keep SKUs Without Open Task` drops SKUs already in an open task's Payload | Always Output Data = ON, Execute Once = ON |
| W4 Sourcing | Supplier email (lowercase) | `Load All Suppliers` → **Merge** `Match Supplier by Email` (`supplierKey` = `ContactEmail`, *Enrich Input 1*) → update if found, else create | Always Output Data = ON, Execute Once = ON |
| Procurement (Phase 2) | `PO-` + ApprovalID → QuickBooks PO DocNumber | `select Id from PurchaseOrder where DocNumber = 'PO-SQ-20261121-101500'` | Same as W2 |
| Payments (Phase 2) | Bank line ID (date + amount + reference hash) → Airtable `PaymentID` | Search Orders where `{PaymentID}` contains the line ID | Same |

> **Why one lookup plus a Merge, not a lookup per item?** If a Search node with "Always Output Data" receives several items, such as 3 emails in one Gmail poll, items with *no* match produce no output and are silently dropped. Looking everything up once and joining with a Merge (Enrich Input 1) keeps every item. Keep this pattern if you build the Phase 2 agents.

### 6.3 Duplicate-prevention pseudocode (invoice and PO)

```
function createInvoiceOnce(order):
    if order.InvoiceID != "" or order.ZRA_InvoiceNo != "":
        log("SkipAlreadyInvoiced", order.OrderID); return order.InvoiceID

    existing = QBO.query("select Id from Invoice where DocNumber = '" + order.OrderID + "'")
    if existing.count > 0:
        Airtable.update(order, InvoiceID = existing[0].Id, Status = "Invoiced")
        telegram("Duplicate prevented for " + order.OrderID)
        log("DuplicatePrevented", order.OrderID); return existing[0].Id

    approval = telegram.sendAndWait(summary(order))         # human gate
    if approval.Decision != "Approve": return null          # empty = NOT approved

    invoice = QBO.create(Invoice, DocNumber = order.OrderID, ...)   # NO automatic retry on this node
    Airtable.update(order, InvoiceID = invoice.Id, Status = "Invoiced")   # write back immediately
    log("CreateQBOInvoice", order.OrderID, approvedBy = approval.Reviewer)
    return invoice.Id

function createPurchaseOrderOnce(approvalRecord):
    poNumber = "PO-" + approvalRecord.ApprovalID           # deterministic, ≤ 21 chars
    if QBO.query("select Id from PurchaseOrder where DocNumber = '" + poNumber + "'").count > 0:
        log("DuplicatePrevented", poNumber); return
    ... same approve → create → write-back → log sequence ...
```

Note in the JSON: **`QBO – Create Invoice` has Retry On Fail switched OFF**. A timeout can occur *after* QuickBooks has created the invoice. A blind retry would then create a second one. On failure, the Error Handler alerts you instead. When the workflow runs again, Check 2 finds the invoice if it exists, so a re-run is always safe.

### 6.4 Retry policy (n8n node **Settings** tab)

| Setting | Value | Applies to |
|---|---|---|
| Retry On Fail | ON | AI Agent, Airtable, Google Sheets, Telegram, QBO *query* nodes |
| Max Tries | **3** (= 1 try + 2 retries) | same |
| Wait Between Tries (ms) | **5000** (the n8n maximum) | same |
| Timeout | **30000 ms** in HTTP Request → Options → Timeout; `timeout: 30000` + `maxRetries: 2` in the OpenAI/DeepSeek model options | QuickBooks HTTP nodes, LLM model nodes |
| Workflow timeout | **300 s**: ⋯ → Settings → Timeout Workflow | all workflows (preset) |
| Retry On Fail | **OFF** | `QBO – Create Invoice`, and any future *create* in QuickBooks |

**Exponential backoff:** n8n's built-in retry waits a fixed time. The LLM model nodes add their own exponential backoff internally (`maxRetries: 2`) for rate-limit errors (HTTP 429). For everything else, a fixed 5-second wait with 2 retries is enough at 10–20 orders a month. If Airtable returns 429 errors (5 requests/second limit), add a **Wait** node of 1 second before the Airtable node.

### 6.5 Dead-letter queue

A "dead-letter queue" is the place failed work goes so it is never silently lost. Here it is the **Airtable Approvals table with Status = Error**.

1. **Workflow crashes** (any node fails after its retries): n8n runs **AIOS – Error Handler**, which:
   - creates an Approvals row: `ApprovalID = ERR-20261121-071502-4411`, `Action = WorkflowError`, `Status = Error`, `Payload = {"workflow":"AIOS – Inventory Watchdog","failedNode":"Read Inventory Sheet","message":"The caller does not have permission","executionUrl":"https://…/execution/4411"}`
   - sends a Telegram 🚨 message with the workflow, failed step, error text and a link
   - appends an AuditLog row with `ErrorFlag = TRUE`
2. **Business-rule failure** (e.g. TPIN missing): W2's `Dead-letter: Approvals (Status=Error)` node writes the issues into `DenialReason`, and Telegram shows ⛔ with the exact problems.
3. **Clearing it:** create an Airtable view **"Dead letters"**: filter `Status = Error`, sorted newest first. Each Monday, fix the cause, then change the row's Status to `Denied` (meaning "handled") and write what you did in DenialReason. The Reporting Agent counts rows that are still `Error`.

To make the Error Handler fire, every workflow must name it: **⋯ → Settings → Error Workflow → AIOS – Error Handler**.
