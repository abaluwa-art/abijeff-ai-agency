## Section 1 — Architecture Diagram

**Key terms:** AIOS = AI Operating System. HITL = Human-in-the-Loop: a person approves before the system acts. ZRA = Zambia Revenue Authority. TPIN = Taxpayer Identification Number, 10 digits. CAT = Central Africa Time (UTC+2). QBO = QuickBooks Online. LLM = Large Language Model, the AI "brain", here GPT-4o-mini or DeepSeek.

```
╔══════════════════════════════════════════════════════════════════════════════════════════╗
║                    AIOS — MUKWA HARDWOODS LTD (Lusaka)  ·  sourcing → payments             ║
╚══════════════════════════════════════════════════════════════════════════════════════════╝

  EVENT SOURCES                          ┌────────────────────────────────────────────────┐
 ┌──────────────────────┐ email.inquiry  │            ORCHESTRATOR — n8n                   │
 │ Gmail label          │───────────────▶│  (runs every workflow, retries, error routing) │
 │ "Inquiries"          │                │                                                │
 ├──────────────────────┤ email.supplier │  ┌──────────────────────────────────────────┐  │
 │ Gmail label          │───────────────▶│  │ SHORT-TERM MEMORY: Window Buffer (last 6  │  │
 │ "Suppliers"          │                │  │ messages per email thread / order)        │  │
 ├──────────────────────┤ cron.07:00     │  └──────────────────────────────────────────┘  │
 │ Schedule (CAT)       │───────────────▶│                                                │
 ├──────────────────────┤ order.ready    │  LLM: GPT-4o-mini (default) · DeepSeek (cheap  │
 │ Airtable Orders      │───────────────▶│       extraction for supplier emails)          │
 │ Status=ReadyToInvoice│                └───────────────┬────────────────────────────────┘
 ├──────────────────────┤ bank.line                      │ dispatches to
 │ QBO bank feed / CSV  │──────────────────────────────▶ │
 └──────────────────────┘                                ▼
 ┌─────────────────────────────────────────── AGENT LAYER ─────────────────────────────────────┐
 │                                                                                              │
 │  ① SOURCING ──quote.parsed──▶ ② PROCUREMENT ──po.draft──▶ [HITL] ──po.approved──▶ QBO Bill   │
 │       ▲                                                                                      │
 │       │ sourcing.task                                                                        │
 │  ③ INVENTORY (daily 07:00) ──stock.low──▶ Telegram briefing                                  │
 │                                                                                              │
 │  ④ SALES/LEADS ──lead.created──▶ Airtable Leads ──reply.draft──▶ Gmail Drafts (human sends)  │
 │                                       │ lead.won → Orders (human builds order + price)       │
 │                                       ▼                                                      │
 │  ⑤ INVOICING ──invoice.pack──▶ [HITL] ──invoice.approved──▶ QBO Invoice ──▶ ZRA Smart Invoice │
 │                                                                   (fiscal invoice, by human) │
 │  ⑥ PAYMENTS ──match.proposed──▶ [HITL] ──payment.approved──▶ QBO Receive Payment            │
 │                                                                                              │
 │  ⑦ RECONCILIATION (Mon) ──recon.report──▶ overdue AR + missing fiscal invoices ──▶ Owner     │
 │  ⑧ REPORTING (Fri 16:00) ──weekly.report──▶ Gmail to Owner                                   │
 └───────────────┬───────────────────────────────┬────────────────────────────┬─────────────────┘
                 │ state.read / state.write      │ approval.request           │ audit.append
                 ▼                               ▼                            ▼
 ┌───────────────────────────────┐ ┌──────────────────────────────┐ ┌───────────────────────────┐
 │ LONG-TERM MEMORY / STATE      │ │ HITL GATE — Telegram bot      │ │ AUDIT LOG (append-only)    │
 │ Airtable base "AIOS_OPS"      │ │  ✅ Approve  ❌ Deny           │ │ Google Sheets "AuditLog"   │
 │  Leads · Orders · Suppliers   │ │  📝 Need More Info            │ │ + "TokenCosts"             │
 │  Prompts · Approvals          │ │ Every money-moving step:      │ │ Every action, every agent, │
 │  (Approvals also = dead-      │ │  invoice, PO, bill, payment,  │ │ who approved, cost, errors │
 │   letter queue, Status=Error) │ │  supplier-quote acceptance    │ │ Kept 6 years (ZRA)         │
 └───────────────────────────────┘ └──────────────────────────────┘ └───────────────────────────┘
                 │                                                             ▲
                 ▼ tool.call                                                   │ error.alert
 ┌──────────────────────────────────────── TOOL LAYER ─────────────────────────┴───────────────┐
 │ QuickBooks Online (invoices, bills, payments) · Gmail (read, DRAFT only) · Google Sheets     │
 │ (Inventory = read-only for agents) · ZRA Smart Invoice (fiscal invoices, issued by a human   │
 │ through the certified path) · Telegram (alerts + approvals)                                   │
 └──────────────────────────────────────────────────────────────────────────────────────────────┘
 ┌──────────────────────────────────────────────────────────────────────────────────────────────┐
 │ ERROR PATH: any node fails → retry ×2 → "AIOS – Error Handler" → Airtable Approvals           │
 │ (Status=Error) → Telegram 🚨 alert → AuditLog row with ErrorFlag=TRUE                          │
 └──────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Design rules shown in the diagram

1. **Agents think. Deterministic nodes act.** Agents read, classify and recommend, then return structured JSON. Ordinary n8n nodes do every write to Airtable, QuickBooks or Gmail, after checks and approvals. Example: the Invoicing Agent never calls QuickBooks. An HTTP node does, and only after you tap Approve.
2. **Code does the maths, never the LLM.** VAT at 16% on ZMW 112,455.00 is ZMW 17,992.80. A Code node calculates that, and the agent only checks that it is plausible.
3. **Nothing reaches a customer without a human.** Replies are Gmail **drafts**. Invoices reach customers only as **ZRA fiscal invoices** that you issue.
4. **One source of truth per thing.** Airtable holds workflow state, Google Sheets holds stock and the audit trail, and QuickBooks holds the books.
5. **Read-only tools for agents.** The Sales Agent can read Inventory, and the Sourcing Agent can read Suppliers. Neither can write.

### Agent summary

| # | Agent | Trigger | Writes to | HITL (Telegram) | MVP? |
|---|-------|---------|-----------|-----------------|------|
| 1 | Sourcing | Gmail label "Suppliers" | Suppliers, Approvals | Approve supplier quote | ✅ W4 |
| 2 | Procurement | Approvals: ApproveSupplierQuote | QBO Purchase Order / Bill | Every PO and bill | Phase 2 |
| 3 | Inventory | Daily 07:00 CAT | Approvals (SourcingTask) | None (read-only) | ✅ W3 |
| 4 | Sales/Leads | Gmail label "Inquiries" | Leads, Gmail Drafts | Owner sends each draft | ✅ W1 |
| 5 | Invoicing | Orders Status = ReadyToInvoice | QBO Invoice, Orders | Every invoice | ✅ W2 |
| 6 | Payments | Bank feed / statement CSV | QBO Payment | Every match | Phase 2 |
| 7 | Reconciliation | Monday 08:00 CAT | Report only | None (report) | Phase 2 |
| 8 | Reporting | Friday 16:00 CAT | Email to owner | None (report) | Week 10 |
