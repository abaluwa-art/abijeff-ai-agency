## Section 2 — Data Schemas

Every table below also exists as a CSV file in `templates/`. The fastest setup is **Airtable → + Add or import → CSV file**, and **Google Sheets → File → Import → Upload → Insert new sheet(s)**. After importing, correct the field types to match the tables below. Airtable guesses types from CSV, and it often guesses wrong.

> **Fields added to the brief.** The workflows need a few fields the original list did not include. Each one is marked ➕ below:
> - **Leads.SourceMessageID** stops a single email from creating two leads. **Leads.EmailReceivedAt** feeds the response-time KPI.
> - **Orders.Email, QBO_CustomerID, InvoiceCurrency, FXRate and ZRA_InvoiceNo** are needed to create the QuickBooks invoice and to record the ZRA fiscal number.
> - **Approvals Status** also takes `NeedsInfo` and `Error`, so the table can serve as the dead-letter queue.

### A. Airtable base `AIOS_OPS`

Create it at **airtable.com → + Create → Start from scratch → name it `AIOS_OPS`**. Then create five tables.

#### Table `Leads`

| Field | Airtable type | Options / format | Example |
|---|---|---|---|
| LeadID (primary) | Single line text | Written by n8n | `LEAD-20261118-091502-a1f3` |
| DateCreated | Date | Include time, GMT+2 (Africa/Lusaka) | `2026-11-18 09:15` |
| CustomerName | Single line text | | `Chanda Mwale` |
| Company | Single line text | | `Mwale Joinery Ltd` |
| Email | Email | | `chanda@mwalejoinery.co.zm` |
| Phone | Phone number | | `+260 977 123456` |
| Species | Single select | Rosewood, Teak, Mahogany, Kiaat, Multiple, Unknown | `Teak` |
| Volume_m3 | Number | Decimal, 2 places | `6.50` |
| Grade | Single line text | | `Furniture-grade KD` |
| DeliveryLocation | Single line text | | `Kitwe` |
| Timeline | Single line text | | `Within 3 weeks` |
| TPIN | Single line text | **Text, not Number**, or leading zeros are lost | `1002345678` |
| Status | Single select | New, Qualified, Quoted, Won, Lost | `Qualified` |
| AssignedAgent | Single line text | | `Sales Agent v1` |
| Notes | Long text | | `[genuine] Joinery in Kitwe wants…` |
| ➕ SourceMessageID | Single line text | Gmail message ID (idempotency key) | `18c2f0a9b7e4a1f3` |
| ➕ EmailReceivedAt | Date | Include time. When the email arrived (used for the response-time KPI) | `2026-11-18 09:14` |

#### Table `Orders`

| Field | Airtable type | Options / format | Example |
|---|---|---|---|
| OrderID (primary) | Single line text | You type it: `ORD-YYYY-NNNN` (max 21 characters, because it becomes the QuickBooks DocNumber) | `ORD-2026-0147` |
| LeadID | Single line text | | `LEAD-20261118-091502-a1f3` |
| CustomerName | Single line text | Must match the QuickBooks customer | `Chongwe Builders Ltd` |
| ➕ Email | Email | | `accounts@chongwebuilders.co.zm` |
| TPIN | Single line text | 10 digits | `1003456789` |
| ➕ QBO_CustomerID | Single line text | QuickBooks → Customers → open customer → number at the end of the URL (`nameId=58`) | `58` |
| ➕ InvoiceCurrency | Single select | ZMW, USD | `ZMW` |
| ➕ FXRate | Number | Decimal, 4 places. ZMW per 1 USD on the invoice date (Bank of Zambia mid-rate) | `25.5000` |
| LineItems | Long text | JSON. **Prices are always in USD per m³**; the workflow converts them | see below |
| TotalUSD | Currency | $ symbol, 2 places | `5,115.60` |
| TotalZMW | Currency | Custom symbol `K`, 2 places | `130,447.80` |
| Status | Single select | Quote, Draft, Confirmed, Production, ReadyToInvoice, Invoiced, Paid | `ReadyToInvoice` |
| DeliveryConfirmed | Checkbox | | ✔ |
| InvoiceID | Single line text | QuickBooks invoice Id (filled by n8n) | `1042` |
| ➕ ZRA_InvoiceNo | Single line text | Fiscal invoice number from ZRA Smart Invoice (filled by you) | `ZRA-SI-0000123456` |
| PaymentID | Single line text | QuickBooks payment Id | `2211` |
| CreatedAt | Created time | | `2026-11-10 10:00` |
| UpdatedAt | **Last modified time** | All fields. **Required by the Airtable trigger** | `2026-11-20 16:45` |

`LineItems` example. Copy it exactly, with straight quotes and no line breaks needed:

```json
[{"sku":"TEAK-KD-50x150","species":"Teak","grade":"Furniture-grade KD","description":"Teak KD 50x150mm, 2.4-4.8m","qty_m3":4.5,"unit_price_usd":980}]
```

Two-line example (ORD-2026-0148): Rosewood 2.0 m³ × USD 1,450 plus Mahogany 3.0 m³ × USD 1,050 gives a subtotal of USD 6,050.00, VAT 16% of USD 968.00, and a total of USD 7,018.00.

```json
[{"sku":"ROS-KD-50x150","species":"Rosewood","grade":"Select","description":"Rosewood KD 50x150mm","qty_m3":2.0,"unit_price_usd":1450},{"sku":"MAH-KD-38x200","species":"Mahogany","grade":"FAS","description":"Mahogany KD 38x200mm","qty_m3":3.0,"unit_price_usd":1050}]
```

#### Table `Suppliers`

| Field | Airtable type | Options / format | Example |
|---|---|---|---|
| SupplierID (primary) | Single line text | | `SUP-001` |
| Name | Single line text | | `Kabompo Timber Concessions` |
| Species | Single line text | Comma-separated | `Kiaat, Rosewood` |
| LastPrice_m3 | Currency | USD, 2 places | `580.00` |
| Reliability | Rating | 5 stars (1–5) | ★★★★ |
| ContactEmail | Email | **Lowercase** (used for matching) | `sales@kabompotimber.co.zm` |
| LastContactDate | Date | ISO (2026-11-12) | `2026-11-12` |
| Notes | Long text | Record licence / conveyance-licence status here | `Licensed concession…` |

#### Table `Prompts` (version control for agent instructions)

| Field | Airtable type | Example |
|---|---|---|
| PromptID (primary) | Single line text | `PRM-SALES-2` |
| AgentName | Single select: the 8 agent names | `Sales Agent` |
| Version | Number (integer) | `2` |
| PromptText | Long text | Full text from `prompts/sales-agent.md` |
| DateUpdated | Date | `2026-12-03` |
| ChangeReason | Long text | `Agent was quoting stock in m³; now says in stock / limited only` |

Rule: whenever you edit a system prompt in n8n, add a new row here first, with Version + 1 and the reason. Never overwrite an old row. That history is how you find out why accuracy changed.

#### Table `Approvals` (HITL log + dead-letter queue)

| Field | Airtable type | Options / format | Example |
|---|---|---|---|
| ApprovalID (primary) | Single line text | `APR-…` invoice, `SQ-…` supplier quote, `SRC-…` sourcing task, `ERR-…` error | `APR-20261120-164800-ORD-2026-0147` |
| OrderID | Single line text | | `ORD-2026-0147` |
| AgentName | Single line text | | `Invoicing Agent` |
| Action | Single select | PrepareInvoice, ApproveSupplierQuote, SourcingTask, CreatePO, CreateBill, ApplyPayment, WorkflowError | `PrepareInvoice` |
| Payload | Long text | JSON snapshot of exactly what was approved | `{"total":130447.8,…}` |
| Status | Single select | Pending, Approved, Denied, ➕ NeedsInfo, ➕ Error | `Approved` |
| Reviewer | Single line text | | `Owner` |
| ReviewedAt | Date | Include time | `2026-11-20 16:52` |
| DenialReason | Long text | | `Wrong FX rate; use 25.62` |

> ⚠️ **Airtable plan.** The Free plan allows about **1,000 API calls a month**. The Invoice trigger polls every 15 minutes, which is about 2,900 calls a month on its own. Buy **Airtable Team (1 seat)**. It is in the budget (Section 13). On Free, the workflows stop working partway through the month.

### B. Google Sheets `Inventory`

Create one spreadsheet called **`AIOS Sheets`** with four tabs: `Inventory`, `AuditLog`, `TokenCosts`, `Dashboard`. If your existing inventory sheet lives elsewhere, move it into this spreadsheet. Keeping everything in one place makes the n8n setup simpler.

| Column | Format | Notes |
|---|---|---|
| SKU | Text | Unique. Pattern `SPECIES-DRYING-THICKxWIDTH` |
| Species | Text: Rosewood / Teak / Mahogany / Kiaat | Data validation: **Data → Data validation → Dropdown** |
| Grade | Text | FAS, Select, No.1 Common, Furniture-grade KD |
| Dimensions | Text | mm × mm × length range |
| Stock_m3 | Number, 1 decimal | |
| ReorderLevel_m3 | Number, 1 decimal | |
| LastUpdated | **Date, format `yyyy-mm-dd`** | **Format → Number → Custom date and time → `yyyy-mm-dd`**. Other formats are treated as stale |
| Location | Text | |
| Notes | Text | |

Sample rows (also in `templates/sheets-Inventory.csv`):

| SKU | Species | Grade | Dimensions | Stock_m3 | ReorderLevel_m3 | LastUpdated | Location | Notes |
|---|---|---|---|---|---|---|---|---|
| TEAK-KD-50x150 | Teak | Furniture-grade KD | 50 x 150 mm x 2.4-4.8 m | 18.4 | 8.0 | 2026-11-21 | Yard A – Kiln shed | KD to 10-12% MC |
| ROS-KD-50x150 | Rosewood | Select | 50 x 150 mm x 1.8-3.6 m | 1.2 | 4.0 | 2026-11-20 | Yard A – Rack 3 | Mukusi; slow-drying |
| MAH-KD-38x200 | Mahogany | FAS | 38 x 200 mm x 2.1-4.2 m | 6.7 | 5.0 | 2026-11-21 | Yard B – Shed 1 | |
| KIA-AD-75x225 | Kiaat | Select | 75 x 225 mm x 2.4-3.6 m | 9.0 | 5.0 | 2026-11-10 | Yard B – Open stack | Air-dried; recount due |
| TEAK-SAWN-100x100 | Teak | No.1 Common | 100 x 100 mm x 3.0 m | 22.5 | 10.0 | 2026-11-21 | Yard C – Log yard | Green sawn posts |

Expected result with these rows on 2026-11-22: ROS-KD-50x150 is flagged **LOW** (1.2 ≤ 4.0, suggested order 7.0 m³) and KIA-AD-75x225 is flagged **STALE** (last counted 12 days ago).

MC = moisture content.

### C. Google Sheets `AuditLog`

Row 1 headers, exactly as written here:

```
Timestamp | Agent | Action | RecordID | InputSummary | OutputSummary | HumanApproved | ApprovedBy | TokenCost | ErrorFlag
```

| Column | Format | Example |
|---|---|---|
| Timestamp | Plain text `yyyy-MM-dd HH:mm:ss` (CAT) | `2026-11-20 16:53:10` |
| Agent | Text | `Invoicing Agent` |
| Action | Text | `CreateQBOInvoice` |
| RecordID | Text | `ORD-2026-0147` |
| InputSummary | Text | `ZMW 130447.8` |
| OutputSummary | Text | `QBO Invoice Id 1042` |
| HumanApproved | TRUE / FALSE | `TRUE` |
| ApprovedBy | Text | `Owner` |
| TokenCost | Number, 6 decimals (USD) | `0.000412` |
| ErrorFlag | TRUE / FALSE | `FALSE` |

Protect it: **Data → Protect sheets and ranges → AuditLog → Restrict who can edit → Only you**. n8n appends as you, so it keeps working, while staff can no longer edit history. Once a year, **File → Make a copy** into a folder named `AIOS Audit Archive YYYY`. Section 7 covers the 6-year retention rule.

### D. Google Sheets `TokenCosts`

```
Timestamp | Agent | Model | InputTokens | OutputTokens | CostUSD | OrderID
```

Example row: `2026-11-20 16:53:10 | Invoicing Agent | gpt-4o-mini | 1180 | 210 | 0.000303 | ORD-2026-0147`

These figures are **estimates**: characters divided by 4, priced at USD 0.15 per million input tokens and USD 0.60 per million output tokens for GPT-4o-mini. Every Friday, compare them with the real bill at **platform.openai.com → Usage**. If the real bill is more than twice the estimate, see Troubleshooting #9.
