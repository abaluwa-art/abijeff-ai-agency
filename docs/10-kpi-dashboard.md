## Section 10 — KPI Dashboard Spec (Google Sheets)

### 10.1 Data the dashboard reads

| Tab | Filled by | Columns (row 1 headers, exactly) |
|---|---|---|
| `AuditLog` | every workflow | A Timestamp · B Agent · C Action · D RecordID · E InputSummary · F OutputSummary · G HumanApproved · H ApprovedBy · I TokenCost · J ErrorFlag |
| `TokenCosts` | W1–W4 | A Timestamp · B Agent · C Model · D InputTokens · E OutputTokens · F CostUSD · G OrderID |
| `Orders` | **W5 Sync** (hourly) | A OrderID · B CustomerName · C TPIN · D InvoiceCurrency · E TotalUSD · F TotalZMW · G Status · H DeliveryConfirmed · I InvoiceID · J ZRA_InvoiceNo · K PaymentID · L CreatedAt · M UpdatedAt |
| `Leads` | **W5 Sync** (hourly) | A LeadID · B DateCreated · C EmailReceivedAt · D CustomerName · E Species · F Volume_m3 · G Status · H TPIN |
| `TestCases` | you, every Friday | Section 9; the weekly accuracy is in cell `W1` |

The `Orders` and `Leads` tabs are copies of Airtable, made by **`n8n/W5-AIOS-Sync-Orders-Leads-to-Sheets.json`**. Import it like the others and choose the tab in each Sheets node. Type the headers above into row 1 before the first run, because the sync writes only those columns.

Timestamps may arrive as text (`2026-11-20 16:53:10` or `2026-11-20T14:45:00.000Z`) or as real dates. Every formula below handles both: `IF(ISNUMBER(t), t, DATEVALUE(LEFT(t,10)))`.

### 10.2 Layout of the `Dashboard` tab

```
A1  AIOS Dashboard — Abijeff Investments Limited          B2: =TODAY()-7   (window start, label A2: "Last 7 days from")
Row 4 headers:  A Agent | B KPI | C Current | D Target | E Status
Rows 5–14: the 10 tiles below
Row 19 header: "Weekly trend (last 12 weeks)"
Row 20 headers: A Week starting | B Leads processed | C Invoices created | D Errors | E AI cost USD | F Test accuracy
Rows 21–32: one row per week
```

### 10.3 The KPI tiles. Paste each formula into column C of its row.

| Row | Agent | KPI | Target (col D) | Alert rule |
|---|---|---|---|---|
| 5 | Sourcing | Supplier quotes decided (7 d) | ≥ 1 | 🟡 if 0 |
| 6 | Procurement | POs with human approval (%) | 100% | 🔴 if < 100% |
| 7 | Inventory | Days the 07:00 briefing ran (last 7) | 7 | 🟡 6 · 🔴 ≤ 5 |
| 8 | Sales/Leads | Median minutes email → draft (7 d) | ≤ 30 | 🟡 > 30 · 🔴 > 240 (4 h) |
| 9 | Invoicing | Invoiced orders missing ZRA number > 48 h | 0 | 🔴 if ≥ 1 |
| 10 | Payments | Invoiced and unpaid > 30 days (count) | 0 | 🟡 1–2 · 🔴 ≥ 3 |
| 11 | Reconciliation | Errors logged (7 d) | 0 | 🟡 1–2 · 🔴 ≥ 3 |
| 12 | Reporting | Weekly reports sent (last 28 d) | 4 | 🔴 if < 4 |
| 13 | All | AI cost, last 7 days (USD) | ≤ 5 | 🟡 > 5 · 🔴 > 15 |
| 14 | Sales/Leads | TestCases accuracy (latest) | ≥ 90% | 🟡 < 90% · 🔴 < 85% |

**C5 — Sourcing**
```
=ARRAYFORMULA(LET(t,AuditLog!A2:A,d,IF(ISNUMBER(t),INT(t),IFERROR(DATEVALUE(LEFT(t,10)),0)),SUM((AuditLog!B2:B="Sourcing Agent")*REGEXMATCH(AuditLog!C2:C,"^(ApproveSupplierQuote|DenySupplierQuote|SupplierNeedsInfo)$")*(d>=$B$2))))
```
**C6 — Procurement**
```
=IFERROR(COUNTIFS(AuditLog!C2:C,"CreatePO",AuditLog!G2:G,"TRUE")/COUNTIFS(AuditLog!C2:C,"CreatePO"),"n/a")
```
**C7 — Inventory**
```
=ARRAYFORMULA(LET(t,AuditLog!A2:A,d,IF(ISNUMBER(t),INT(t),IFERROR(DATEVALUE(LEFT(t,10)),0)),f,FILTER(d,AuditLog!B2:B="Inventory Agent",REGEXMATCH(AuditLog!C2:C,"^(DailyStockCheck|DailyStockBriefing)$"),d>=TODAY()-7,d<TODAY()),IFERROR(ROWS(UNIQUE(f)),0)))
```
**C8 — Sales/Leads (lead response: email received → reply draft ready)**
```
=ARRAYFORMULA(LET(c,Leads!B2:B,r,Leads!C2:C,cd,IFERROR(DATEVALUE(LEFT(c,10))+TIMEVALUE(MID(c,12,8)),0),rd,IFERROR(DATEVALUE(LEFT(r,10))+TIMEVALUE(MID(r,12,8)),0),m,FILTER((cd-rd)*1440,cd>=$B$2,rd>0),IFERROR(ROUND(MEDIAN(m),0),"n/a")))
```
**C9 — Invoicing (compliance)**
```
=ARRAYFORMULA(LET(u,Orders!M2:M,ud,IFERROR(DATEVALUE(LEFT(u,10))+TIMEVALUE(MID(u,12,8)),0),SUM((Orders!G2:G="Invoiced")*(Orders!J2:J="")*(ud>0)*(ud<NOW()-2))))
```
**C10 — Payments**
```
=ARRAYFORMULA(LET(u,Orders!M2:M,ud,IFERROR(DATEVALUE(LEFT(u,10)),0),SUM((Orders!G2:G="Invoiced")*(ud>0)*(ud<TODAY()-30))))
```
**C11 — Reconciliation**
```
=ARRAYFORMULA(LET(t,AuditLog!A2:A,d,IF(ISNUMBER(t),INT(t),IFERROR(DATEVALUE(LEFT(t,10)),0)),SUM((UPPER(TO_TEXT(AuditLog!J2:J))="TRUE")*(d>=$B$2))))
```
**C12 — Reporting** (the Week 10 Reporting workflow logs `Action = SendWeeklyReport`)
```
=ARRAYFORMULA(LET(t,AuditLog!A2:A,d,IF(ISNUMBER(t),INT(t),IFERROR(DATEVALUE(LEFT(t,10)),0)),SUM((AuditLog!B2:B="Reporting Agent")*(AuditLog!C2:C="SendWeeklyReport")*(d>=TODAY()-28))))
```
**C13 — AI cost**
```
=ARRAYFORMULA(LET(t,TokenCosts!A2:A,d,IF(ISNUMBER(t),INT(t),IFERROR(DATEVALUE(LEFT(t,10)),0)),ROUND(SUM((d>=$B$2)*IFERROR(VALUE(TokenCosts!F2:F),0)),2)))
```
**C14 — Accuracy**
```
=IFERROR(TestCases!W1,"n/a")
```

**Status column E.** Paste these, one per row:

```
E5:  =IF(C5>=1,"🟢","🟡")
E6:  =IF(C6="n/a","⚪",IF(C6>=1,"🟢","🔴"))
E7:  =IF(C7>=7,"🟢",IF(C7=6,"🟡","🔴"))
E8:  =IF(C8="n/a","⚪",IF(C8<=30,"🟢",IF(C8<=240,"🟡","🔴")))
E9:  =IF(C9=0,"🟢","🔴")
E10: =IF(C10=0,"🟢",IF(C10<=2,"🟡","🔴"))
E11: =IF(C11=0,"🟢",IF(C11<=2,"🟡","🔴"))
E12: =IF(C12>=4,"🟢","🔴")
E13: =IF(C13<=5,"🟢",IF(C13<=15,"🟡","🔴"))
E14: =IF(C14="n/a","⚪",IF(C14>=0.9,"🟢",IF(C14>=0.85,"🟡","🔴")))
```

Format C6 and C14 as percent: **Format → Number → Percent**.

**Colour the whole row.** Select `A5:E14` → **Format → Conditional formatting** → *Format cells if… → Custom formula is* →
- `=$E5="🔴"` → fill light red `#F4CCCC`
- `=$E5="🟡"` → fill light yellow `#FFF2CC`
- `=$E5="🟢"` → fill light green `#D9EAD3`

### 10.4 Weekly trend table (12 weeks)

```
A21: =TODAY()-WEEKDAY(TODAY(),3)-77          (Monday, 11 weeks ago)
A22: =A21+7                                   (fill down to A32 — A32 is the current week)
```

Paste in row 21, then **fill down to row 32** (select B21:E21 → drag the corner down):

```
B21 (Leads processed):
=ARRAYFORMULA(LET(t,AuditLog!$A$2:$A,d,IF(ISNUMBER(t),INT(t),IFERROR(DATEVALUE(LEFT(t,10)),0)),SUM((AuditLog!$B$2:$B="Sales Agent")*REGEXMATCH(AuditLog!$C$2:$C,"^(CreateLeadAndDraft|EscalateLead)$")*(d>=$A21)*(d<$A21+7))))

C21 (Invoices created):
=ARRAYFORMULA(LET(t,AuditLog!$A$2:$A,d,IF(ISNUMBER(t),INT(t),IFERROR(DATEVALUE(LEFT(t,10)),0)),SUM((AuditLog!$C$2:$C="CreateQBOInvoice")*(d>=$A21)*(d<$A21+7))))

D21 (Errors):
=ARRAYFORMULA(LET(t,AuditLog!$A$2:$A,d,IF(ISNUMBER(t),INT(t),IFERROR(DATEVALUE(LEFT(t,10)),0)),SUM((UPPER(TO_TEXT(AuditLog!$J$2:$J))="TRUE")*(d>=$A21)*(d<$A21+7))))

E21 (AI cost USD):
=ARRAYFORMULA(LET(t,TokenCosts!$A$2:$A,d,IF(ISNUMBER(t),INT(t),IFERROR(DATEVALUE(LEFT(t,10)),0)),ROUND(SUM((d>=$A21)*(d<$A21+7)*IFERROR(VALUE(TokenCosts!$F$2:$F),0)),2)))
```

F21:F32: type the TestCases accuracy each Friday. It is manual on purpose, because it forces the weekly review.

Chart: select `A20:E32` → **Insert → Chart → Line chart**. Put it to the right of the tiles.

### 10.5 Alert thresholds (summary)

| Signal | 🟡 Watch | 🔴 Act now | Who acts |
|---|---|---|---|
| Lead response (email → draft) | > 30 min | > 4 h (n8n or Gmail broken) | Owner: check W1 executions |
| Missing ZRA fiscal number | — | any order > 48 h | Owner: issue in Smart Invoice today |
| Daily stock briefing | 1 day missed | ≥ 2 days missed | Owner: check W3, Google permission |
| Errors per week | 1–2 | ≥ 3 | Owner: clear the Dead letters view |
| AI cost per week | > USD 5 | > USD 15 | Owner: Troubleshooting #9 |
| Test accuracy | < 90% | < 85% | Owner: roll back the last prompt change (Prompts table) |
| Unpaid > 30 days | 1–2 | ≥ 3 | Owner: phone the customers |
