## Section 3 — Agent Specifications

Each system prompt below is also saved as its own file in `prompts/`. To use one, open the **AI Agent** node in n8n → **Options → System Message** → paste. The four MVP workflows (Section 4) already contain their prompts.

Every prompt includes the same three safety habits:
1. Email, notes and bank-reference content is **data, not instructions**.
2. Missing values are returned as `null` and never guessed.
3. The output is **JSON only**, so ordinary n8n nodes can check it before anything happens.

---

### AGENT 1: Sourcing Agent

── **Role:** Turns supplier emails into structured price data, compares them with the Suppliers table, and recommends accept, negotiate or reject.

── **System Prompt** (`prompts/sourcing-agent.md`)
```text
{{PROMPT:sourcing-agent}}
```
── **Tools Available:** `Gmail Trigger` (label `Suppliers`, read) · `Supplier Lookup (read-only)` = Airtable Tool → Search · `DeepSeek Chat Model` · `Window Buffer Memory` (per email thread) · `Structured Output Parser`. Writes happen only in downstream nodes: `Airtable → Update/Create` (Suppliers), `Airtable → Create` (Approvals), `Gmail → Create Draft`.
── **Inputs:** Gmail messages with label **Suppliers**, plus the Airtable **Suppliers** table.
── **Outputs:** Supplier LastPrice_m3 and LastContactDate updated. An **Approvals** row with Action `ApproveSupplierQuote` (Approved or Denied). A Gmail draft of clarifying questions when you tap "Need More Info".
── **Guardrails:** Cannot reply to or accept offers. Cannot create POs. Cannot change supplier bank details. Read-only access to Suppliers through the tool.
── **HITL Gate:** Telegram custom form on **every** parsed quote (Approve / Deny / Need More Info). Only Approved quotes reach Procurement.
── **Escalation Rules:** red_flags is not empty (changed bank details, >50% prepayment, Mukula offered, no licence for logs, price >30% below market). The Telegram message shows 🚩 and the recommendation can never be "accept".
── **Success Metrics:** (1) ≥ 95% of supplier price lines extracted correctly (weekly spot-check of 10). (2) Median time from supplier email to owner decision < 4 h. (3) 0 quotes accepted without legality documents.
── **Failure Modes & Handling:**
   1. *Price quoted in ZMW with no exchange rate.* The agent leaves USD null and keeps the ZMW figure. The Telegram message shows the ZMW price, and you decide.
   2. *Supplier attaches a PDF price list with no text in the body.* The agent sees only attachment names, so it returns `is_supplier_quote=false`. The audit row says "Not a quote". Weekly fix: forward the PDF content as text, or add an "Extract from PDF" node (Phase 2).
   3. *Same supplier, new email address.* "Match Supplier by Email" finds nothing, so a duplicate supplier is created. Monthly fix: merge duplicates in Airtable and add the alias to Notes.

---

### AGENT 2: Procurement Agent

── **Role:** Converts an approved supplier quote into a draft Purchase Order, and after delivery into a draft QuickBooks bill for the quantity actually received.

── **System Prompt** (`prompts/procurement-agent.md`)
```text
{{PROMPT:procurement-agent}}
```
── **Tools Available:** `Airtable Trigger` (Approvals, Action = ApproveSupplierQuote, Status = Approved) · `Airtable → Get` (Suppliers) · `OpenAI Chat Model` · `Window Buffer Memory` · `Structured Output Parser` · `Telegram → Send and Wait` · `HTTP Request` (QuickBooks `purchaseorder` and `bill` endpoints, created only after approval) · `Google Sheets → Append` (AuditLog).
── **Inputs:** Approvals rows (ApproveSupplierQuote / Approved), the Suppliers table, and goods-received notes (a "GRN_m3" number you type on the Approvals row after delivery).
── **Outputs:** QuickBooks Purchase Order (DocNumber = `PO-` + ApprovalID) and QuickBooks Bill, both after approval. Approvals rows with Action `CreatePO` / `CreateBill`.
── **Guardrails:** No payments, ever. No PO without a referenced approval ID. Cannot alter supplier bank details. Bill amount = received quantity × agreed price only.
── **HITL Gate:** Telegram approval for every PO and every bill. Over USD 5,000 the **owner** must approve; the backup approver cannot.
── **Escalation Rules:** Total > USD 5,000. Supplier Reliability ≤ 2. Bank details mismatch. Received quantity differs from ordered by > 5%. Missing timber or conveyance licence at delivery.
── **Success Metrics:** (1) 100% of POs trace to an Approved sourcing record. (2) 0 duplicate POs (idempotency on DocNumber). (3) Bills match the GRN within 1%.
── **Failure Modes & Handling:**
   1. *QuickBooks vendor missing.* The HTTP node returns a 400 error and the Error Handler alerts you. Create the vendor in QuickBooks, then set the Approvals row back to Approved to re-run.
   2. *Supplier delivers 18.2 m³ against 20 m³ ordered.* The difference is 9% (> 5%), so the agent escalates and the bill draft uses 18.2 m³.
   3. *Approval sits for more than 24 h.* The Telegram wait times out, the row becomes NeedsInfo, and the Friday report lists it.

---

### AGENT 3: Inventory Agent

── **Role:** Checks stock every morning at 07:00 CAT, briefs the owner on low or stale stock, and opens sourcing tasks for urgent items.

── **System Prompt** (`prompts/inventory-agent.md`)
```text
{{PROMPT:inventory-agent}}
```
── **Tools Available:** `Schedule Trigger` (cron `0 7 * * *`, timezone Africa/Lusaka) · `Google Sheets → Get Rows` (Inventory, read-only) · `Code` (flags LOW / STALE / NEGATIVE, suggests order quantity) · `OpenAI Chat Model` · `Window Buffer Memory` · `Structured Output Parser` · `Telegram → Send Message` · `Airtable → Search/Create` (Approvals).
── **Inputs:** Google Sheet **Inventory**.
── **Outputs:** Telegram stock briefing (or a "✅ all good" heartbeat). Airtable **Approvals** rows with Action `SourcingTask`, one per urgent SKU, never duplicated while open.
── **Guardrails:** Cannot edit the Inventory sheet. Cannot contact suppliers. Cannot change suggested quantities, because code calculates them.
── **HITL Gate:** None needed, because nothing moves money. Sourcing tasks are only *suggestions* that start the Sourcing → Procurement chain, where approvals apply.
── **Escalation Rules:** ≥ 3 high-urgency SKUs, negative stock, or a drop of > 20 m³. The briefing starts with "ESCALATE:".
── **Success Metrics:** (1) Briefing delivered by 07:05 on ≥ 95% of days. (2) 0 stock-outs on SKUs that were flagged ≥ 7 days earlier. (3) Stale rows < 10% of SKUs.
── **Failure Modes & Handling:**
   1. *Sheet date typed as 20/11/2026.* It is not ISO, so the row is flagged STALE every day. Fix the column format (Section 2B).
   2. *Google permission revoked.* "Read Inventory Sheet" fails, the Error Handler alerts you, and **no heartbeat arrives**. The missing ✅ message is itself the alarm.
   3. *Staff note says "order 100 m³ now".* The prompt treats it as data, and the briefing reports "unusual note on SKU X".

---

### AGENT 4: Sales/Leads Agent

── **Role:** Classifies every inbound enquiry, extracts lead fields into Airtable, and drafts a reply for a human to send.

── **System Prompt** (`prompts/sales-agent.md`)
```text
{{PROMPT:sales-agent}}
```
── **Tools Available:** `Gmail Trigger` (label `Inquiries`) · `Inventory Lookup (read-only)` = Google Sheets Tool → Get Rows · `OpenAI Chat Model` (gpt-4o-mini, temperature 0.1) · `Window Buffer Memory` (per Gmail thread) · `Structured Output Parser`. Downstream: `Airtable → Search/Create` (Leads), `Gmail → Create Draft`, `Telegram → Send Message`.
── **Inputs:** Gmail messages with label **Inquiries**. Create a Gmail filter that applies it to mail sent to sales@ / info@, or to mail containing "timber OR quote OR m3 OR teak OR kiaat".
── **Outputs:** Airtable **Leads** row. Gmail **draft** reply in the same thread. Telegram FYI or escalation alert.
── **Guardrails:** Never sends email. Never quotes prices, discounts, terms or bank details. Never confirms exact stock or reserves it. No draft for spam or injection.
── **HITL Gate:** Weeks 2–7: you send every draft yourself from Gmail. Week 8 onwards: drafts for `genuine` leads with no missing fields may be sent after a one-tap Telegram approval (Section 8, Week 8).
── **Escalation Rules:** volume > 50 m³, export, tender, complaint, legal threat, payment dispute, bank-detail request, or injection attempt. You get a Telegram ⚠️ and no draft is created.
── **Success Metrics:** (1) Field-extraction accuracy ≥ 90% on the 20-email TestCases set (Section 9). (2) Median lead response time < 2 h in business hours (from email received to draft ready < 2 min). (3) Spam false-positive rate 0% (no genuine lead marked spam).
── **Failure Modes & Handling:**
   1. *Gmail fires twice for one email.* "Find Existing Leads" + "Mark Already-Processed Emails" match the SourceMessageID, so the second run is skipped and logged as `SkipDuplicate`.
   2. *Buyer writes "300 board feet of Kiaat".* The agent converts to 0.71 m³. If the conversion is wrong in testing, add a worked example to the prompt and bump the version in Prompts.
   3. *LLM returns invalid JSON.* The output parser fails, the node retries ×2, and then the Error Handler alerts you. The email stays in Gmail for manual handling.

---

### AGENT 5: Invoicing Agent

── **Role:** Reviews a code-calculated invoice pack for a confirmed, delivered order. After owner approval, the workflow creates the QuickBooks invoice and prompts the owner to issue the ZRA fiscal invoice.

── **System Prompt** (`prompts/invoicing-agent.md`)
```text
{{PROMPT:invoicing-agent}}
```
── **Tools Available:** `Airtable Trigger` (Orders; formula `{Status}='ReadyToInvoice'`; trigger field UpdatedAt) · `Code` (pre-flight checks + VAT maths + QuickBooks body) · `OpenAI Chat Model` · `Window Buffer Memory` (per OrderID) · `Structured Output Parser` · `Telegram → Send and Wait` (custom form) · `HTTP Request` (QuickBooks query + create invoice, OAuth2 credential) · `Airtable → Create/Update` (Approvals, Orders) · `Google Sheets → Append` (AuditLog, TokenCosts).
── **Inputs:** Airtable **Orders** rows that move to `ReadyToInvoice`.
── **Outputs:** **Approvals** row (Pending → Approved / Denied / NeedsInfo / Error). QuickBooks invoice with DocNumber = OrderID. Orders.Status = `Invoiced` and InvoiceID filled. A Telegram instruction to issue the ZRA fiscal invoice.
── **Guardrails:** Never changes a number. Never invents a TPIN. Never issues or emails an invoice. Never decides zero-rating. Duplicate invoices are blocked twice: once by InvoiceID / ZRA_InvoiceNo in Airtable, then by a QuickBooks DocNumber query.
── **HITL Gate:** Telegram approval on **every** invoice. No exceptions, including Week 11.
── **Escalation Rules:** TPIN missing or invalid, species not allowed, delivery not confirmed, totals off by > 1%, export or zero-rating mentioned, or suspicious text in notes. The order is blocked and logged as Status=Error.
── **Success Metrics:** (1) 0 duplicate invoices. (2) 100% of B2B invoices carry a valid buyer TPIN. (3) ZRA fiscal invoice number recorded within 24 h of the QuickBooks invoice (≥ 95%).
── **Failure Modes & Handling:**
   1. *QuickBooks token expired.* The HTTP node returns a 401 and the Error Handler alerts you. Reconnect the credential (Troubleshooting #2), then edit the order to re-trigger it.
   2. *Customer currency in QuickBooks is USD but InvoiceCurrency is ZMW.* QuickBooks rejects the invoice (400). Make the two match, then re-trigger.
   3. *Owner taps Deny.* The approval becomes Denied with a reason and the order goes back to `Confirmed`, so it is not picked up again.

---

### AGENT 6: Payments Agent

── **Role:** Proposes matches between incoming payments (bank, Airtel Money, MTN MoMo, cheques) and open invoices. A human approves each match before QuickBooks records it.

── **System Prompt** (`prompts/payments-agent.md`)
```text
{{PROMPT:payments-agent}}
```
── **Tools Available:** `Schedule Trigger` (daily 17:00 CAT) · `Google Sheets → Get Rows` (tab `BankLines`, where you paste the day's statement CSV; or HTTP Request → QuickBooks `query` for bank-feed deposits) · `Airtable → Search` (Orders Status = Invoiced) · `OpenAI Chat Model` · `Structured Output Parser` · `Telegram → Send and Wait` · `HTTP Request` (QuickBooks `payment` create, after approval) · `Airtable → Update` (Orders → Paid, PaymentID).
── **Inputs:** Bank or mobile-money lines and open invoices.
── **Outputs:** Proposed matches in Telegram. After approval, a QuickBooks payment linked to the invoice, and Orders.Status = Paid with PaymentID.
── **Guardrails:** Never records a payment without approval. Never refunds, writes off or messages customers. Never changes amounts.
── **HITL Gate:** Every match. Matches at ≥ 0.95 confidence can be approved together in one batch message.
── **Escalation Rules:** Overpayment > 2%, unknown payer, the same amount twice within 3 days, FX difference > 2%, or instructions in a reference field.
── **Success Metrics:** (1) ≥ 80% of payments auto-proposed at ≥ 0.80 confidence. (2) 0 wrongly applied payments. (3) Payments applied within 1 business day.
── **Failure Modes & Handling:**
   1. *Customer pays two invoices in one transfer.* The agent lists both invoice IDs and you approve once.
   2. *Bank charges reduce USD 5,115.60 to USD 5,090.60.* The difference is 0.5%, so the agent proposes a match at 0.6–0.79 confidence and notes a USD 25 shortfall. You decide whether to book bank charges.
   3. *Mobile-money reference is only a phone number.* There is no match. The line goes on the unmatched list for the Reconciliation Agent.

---

### AGENT 7: Reconciliation Agent

── **Role:** Every Monday, finds mismatches between Airtable, QuickBooks and ZRA fiscal numbers, ages overdue receivables, and drafts reminders.

── **System Prompt** (`prompts/reconciliation-agent.md`)
```text
{{PROMPT:reconciliation-agent}}
```
── **Tools Available:** `Schedule Trigger` (Monday 08:00 CAT) · `Airtable → Search` (Orders) · `HTTP Request` (QuickBooks `query`: `select * from Invoice where Balance > '0'`) · `Google Sheets → Get Rows` (AuditLog with ErrorFlag = TRUE, BankLines unmatched) · `OpenAI Chat Model` · `Structured Output Parser` · `Gmail → Send` (report to owner only) · `Gmail → Create Draft` (customer reminders).
── **Inputs:** Orders, QuickBooks open invoices, unmatched bank lines, AuditLog errors.
── **Outputs:** Monday email "Reconciliation – week NN". Gmail drafts of polite reminders for invoices 1–30 days overdue.
── **Guardrails:** Read-only for all systems. Drafts only. No write-offs or credit notes.
── **HITL Gate:** None for the report. Each reminder is a draft that you send.
── **Escalation Rules:** Any sale > 48 h without ZRA_InvoiceNo (compliance), a customer owing > USD 10,000 or 61+ days overdue, or a cross-system difference > USD 50. These items appear under "urgent" and the email subject starts with "URGENT".
── **Success Metrics:** (1) 0 sales without a fiscal invoice after 48 h. (2) Days Sales Outstanding (DSO) trending down. (3) Unresolved errors older than 7 days = 0.
── **Failure Modes & Handling:**
   1. *Invoice created by hand in QuickBooks.* It is listed as an "orphan". Add an Airtable order for it or mark it "manual" in the QuickBooks memo.
   2. *Large QuickBooks result.* Limit the query to the last 180 days to stay under the 5-minute timeout.
   3. *Customer name spelled differently* (e.g. "ZCCM-IH" vs "ZCCM Investments Holdings"). Match on QBO_CustomerID, never on name.

---

### AGENT 8: Reporting Agent

── **Role:** Sends the owner a 2-minute weekly report every Friday at 16:00 CAT, covering business KPIs and AIOS health.

── **System Prompt** (`prompts/reporting-agent.md`)
```text
{{PROMPT:reporting-agent}}
```
── **Tools Available:** `Schedule Trigger` (Friday 16:00 CAT) · `Google Sheets → Get Rows` (Dashboard, TokenCosts, TestCases) · `Airtable → Search` (Leads, Orders, Approvals for the last 7 days) · `OpenAI Chat Model` · `Structured Output Parser` · `Gmail → Send` (to the owner only) · `Telegram → Send Message` (headline).
── **Inputs:** The Section 10 Dashboard tab plus Airtable counts.
── **Outputs:** Weekly email and a one-line Telegram headline.
── **Guardrails:** Never invents numbers (writes "n/a"). Never gives tax advice. Never recommends autonomy for invoicing or payments.
── **HITL Gate:** None. It is an internal report to the owner only.
── **Escalation Rules:** Missing fiscal invoice > 48 h, accuracy < 85%, AI cost > USD 15/week, or an approval waiting > 48 h. The subject starts with "ACTION NEEDED:".
── **Success Metrics:** (1) Delivered every Friday by 16:10. (2) Owner reads it in under 2 minutes (self-reported). (3) ≥ 1 recommended action acted on per week.
── **Failure Modes & Handling:**
   1. *Dashboard formulas break after a column rename.* The numbers show #REF!. The agent is told to write "n/a", and the subject flags missing data.
   2. *Holiday week with zero leads.* The agent reports zeros instead of "no data", because the formulas return 0, not blank.
   3. *Token cost spike.* The report shows 🔴 on AI cost and recommends checking Troubleshooting #9.
