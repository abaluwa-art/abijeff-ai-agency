## Section 8 — 90-Day Rollout Plan (week by week)

Rhythm: build early in the week, test mid-week, and decide Go or No-Go on Friday after reading the AuditLog. **Shadow mode** means the agent runs and records what it *would* do, while you keep working as before and compare. **Supervised** means the agent's output is used, with your approval at each step. Hours assume one non-technical operator.

### Week 1 — Foundation
- **Goal:** All accounts, tables and credentials exist, and nothing is automated yet.
- **Tasks:**
  1. Sign up for **n8n Cloud Starter** (Section 13). Set Settings → timezone to Africa/Lusaka.
  2. Create the Airtable base `AIOS_OPS` with 5 tables from `templates/airtable-*.csv`, then fix the field types per Section 2. Upgrade to Airtable Team.
  3. Create the `AIOS Sheets` spreadsheet with tabs Inventory, AuditLog, TokenCosts, Dashboard, TestCases. Move your real inventory into the Inventory tab and format LastUpdated as `yyyy-mm-dd`.
  4. Create OpenAI and DeepSeek API keys. Set the OpenAI monthly budget cap to USD 25.
  5. Create the Telegram bot and get your chat ID (Section 5, steps 1–2).
  6. Create all 7 credentials in n8n (Section 4.2). Leave QuickBooks on **Sandbox**.
  7. Import **W0 Error Handler**, connect its credentials and save it.
  8. **Submit the VAT registration** to ZRA (Section 7, item 1). It must be effective before invoices go live in Week 6.
  9. Book a 1-hour meeting with your accountant for Week 4 to go through Section 7.
- **Deliverable:** Empty but connected stack. A manual test of W0 (open it → Test workflow) sends a 🚨 test message to Telegram.
- **Go/No-Go:** Go if every credential shows "Connection tested successfully" and the Telegram test arrives.
- **Time:** 8–10 h.

### Week 2 — Sales Agent in shadow mode
- **Goal:** W1 processes real enquiries into Airtable and Gmail drafts. You still reply manually as usual.
- **Tasks:**
  1. Create the Gmail label `Inquiries` and its filter (Section 4.3).
  2. Import W1, set the Config chat ID and credentials, and set the Error Workflow.
  3. Paste the 20 test emails (Section 9) into the TestCases tab. Send 5 of them to yourself and run the workflow manually for each.
  4. Activate W1.
  5. Every day: compare each Airtable lead and Gmail draft with what you would have done. Mark mistakes in the lead's Notes as `REVIEW: wrong species`. **Do not send the drafts**; reply manually as before.
- **Deliverable:** ≥ 15 real or test leads in Airtable and the AuditLog filling up.
- **Go/No-Go:** Go if field accuracy is ≥ 80% on TestCases, 0 injection emails produced a draft, and 0 genuine leads were marked spam. Otherwise edit the prompt (log it in Prompts) and repeat the week.
- **Time:** 5–6 h.

### Week 3 — Sales Agent supervised live
- **Goal:** You send the agent's drafts after reviewing them.
- **Tasks:**
  1. For each new lead, open Gmail → Drafts, edit if needed, then press **Send**.
  2. Track how much you edit each draft in the lead's Notes: `EDIT: none / light / heavy`.
  3. Run the full 20-email TestCases set once (Friday) and record the score.
  4. Tune the prompt once, based on the most common heavy edit.
- **Deliverable:** Median "email received → reply sent" time under 2 business hours.
- **Go/No-Go:** Go if ≥ 70% of drafts need no or light edits and TestCases ≥ 85%.
- **Time:** 4 h.

### Week 4 — Inventory Agent live (read-only)
- **Goal:** A daily 07:00 stock briefing in Telegram.
- **Tasks:**
  1. Import W3, set credentials and the Error Workflow, and run it once manually.
  2. Set realistic ReorderLevel_m3 values per SKU, e.g. 3 weeks of average sales. Rosewood and Kiaat need more because of 3–6 week lead times.
  3. Activate W3.
  4. Agree with the yard manager: update Stock_m3 and LastUpdated every Friday.
  5. Hold the accountant meeting (Section 7): check progress on the VAT registration and record the Smart Invoice route.
- **Deliverable:** 5 consecutive daily briefings or ✅ heartbeats, plus a signed Section 7 checklist items 1–2.
- **Go/No-Go:** Go if the briefing arrives by 07:05 every day and the flags match your own judgement.
- **Time:** 4 h.

### Week 5 — Invoicing Agent built (no live invoices)
- **Goal:** W2 works end to end against the **QuickBooks Sandbox**.
- **Tasks:**
  1. Create the Intuit developer app and the QuickBooks Sandbox credential (Section 4.4).
  2. In the Sandbox company, create 4 timber items and 2 test customers (one ZMW, one USD), and enable Multicurrency.
  3. Import W2 and fill in the Config IDs (sandbox base URL).
  4. Create 4 test orders in Airtable: (a) valid ZMW, (b) valid USD, (c) TPIN missing, (d) species "Mukula". Set each to ReadyToInvoice.
  5. Expected: (a) and (b) reach the Telegram approval; (c) and (d) are blocked with ⛔ and appear in Approvals as Error.
  6. Approve (a) **twice**: set it to ReadyToInvoice again after the first run. The second run must say "Duplicate prevented".
- **Deliverable:** 2 Sandbox invoices, 0 duplicates, 2 correctly blocked orders.
- **Go/No-Go:** Go only if all 6 expectations pass **and** the accountant has confirmed the Smart Invoice route. While testing in the Sandbox, put a past date in `vatEffectiveDate` so test invoices aren't blocked. Change it back to the real date before Week 6.
- **Time:** 8–10 h. This is the hardest week; Section 13 budgets for 4 h of paid help here.

### Week 6 — Invoicing Agent live with HITL
- **Goal:** Real invoices go through W2, and you issue every fiscal invoice.
- **Tasks:**
  0. **Gate:** the VAT certificate has arrived and its effective date is in W2 Config `vatEffectiveDate`. No certificate means No-Go for live invoicing; keep invoicing manually.
  1. Reconnect the QuickBooks credential as **Production**. Change `qboBaseUrl` to `https://quickbooks.api.intuit.com` and the Config IDs to the production IDs.
  2. Activate W2.
  3. For every approval: check the Telegram numbers against the order, Approve, issue the fiscal invoice in Smart Invoice, and paste ZRA_InvoiceNo into Airtable.
  4. Friday: tick Section 7 item 3 against the first real fiscal invoice.
- **Deliverable:** All of this week's invoices created through W2 with a ZRA number recorded.
- **Go/No-Go:** Go if 0 wrong invoices, 0 duplicates, and every ZRA number recorded within 24 h. **Any error means No-Go**: fix it, and repeat the week.
- **Time:** 3 h plus 5 min per invoice.

### Week 7 — Sourcing Agent shadow mode
- **Goal:** W4 parses supplier emails and you compare its results with your own reading.
- **Tasks:**
  1. Fill the Suppliers table with your real suppliers, including Reliability and licence notes.
  2. Create the Gmail label `Suppliers` and its filter. Import W4 and set it up.
  3. Activate W4. This week, **answer every approval with "Deny" and Notes "shadow"**, so nothing goes to Procurement while you check accuracy.
  4. Check each extracted price against the email.
- **Deliverable:** ≥ 5 supplier emails parsed, with a price-accuracy score.
- **Go/No-Go:** Go if ≥ 95% of price lines are correct and every bank-detail change was flagged 🚩.
- **Time:** 4 h.

### Week 8 — Sales Agent supervised autonomy
- **Goal:** Simple, complete enquiries are answered with one tap instead of opening Gmail.
- **Tasks:**
  1. In W1, after **Create Gmail Draft Reply**, add a **Telegram → Send and Wait** (Response Type: Approval, Approve and Disapprove) showing the draft text, followed by an IF on `{{ $json.data.approved }}` and a **Gmail → Draft → Send** node. Use this path only when `category = genuine` and `missing_fields` is empty. Everything else stays draft-only.
  2. Keep escalations exactly as they are.
- **Deliverable:** ≥ 50% of genuine leads answered from Telegram in under 15 minutes.
- **Go/No-Go:** Go if 0 wrong replies were sent and TestCases ≥ 90%.
- **Time:** 4 h.

### Week 9 — Sourcing Agent live with HITL
- **Goal:** Approved supplier quotes become Approvals records for Procurement.
- **Tasks:**
  1. Answer W4 approvals for real: Approve, Deny or Need More Info.
  2. Check that "Need More Info" produces a sensible supplier draft and send it.
  3. Build the Procurement Agent as a copy of W2's pattern if you are ready (optional; otherwise create POs by hand from the Approved rows).
- **Deliverable:** First Approved supplier quote leading to a purchase.
- **Go/No-Go:** Go if decisions take you under 2 minutes each and no red flag was missed.
- **Time:** 4 h.

### Week 10 — Reporting Agent live
- **Goal:** A Friday 16:00 email with all KPIs.
- **Tasks:**
  1. Import W5 (sync) and build the Dashboard tab (Section 10).
  2. Create the Reporting workflow: Schedule (cron `0 16 * * 5`) → Sheets Get Rows (Dashboard) → AI Agent with `prompts/reporting-agent.md` and output parser → Gmail Send to you → AuditLog with `Agent = Reporting Agent`, `Action = SendWeeklyReport`. Copy the agent, model and parser nodes from W3 to save time.
  3. Read the first report and check 5 numbers by hand.
- **Deliverable:** First weekly report received.
- **Go/No-Go:** Go if all 5 spot-checked numbers match.
- **Time:** 5 h.

### Week 11 — Optimise and pick one workflow for full autonomy
- **Goal:** Reduce cost and clicks. Promote exactly one *non-financial* step to autonomous.
- **Tasks:**
  1. Review 10 weeks of AuditLog: which approvals were always "Approve" with no edits?
  2. Candidate (recommended): **W3 Inventory sourcing tasks**, already autonomous, or **W1 replies to complete genuine leads where the stock note is "in stock"**. Invoicing and payments **never** become autonomous.
  3. Reduce costs: if TokenCosts > USD 10/week, shorten the email cap from 6,000 to 3,000 characters.
  4. Re-run TestCases. Accuracy must not drop.
- **Deliverable:** A written decision: "From Week 12, [workflow] runs without approval when [conditions]", plus a rollback rule ("if 1 error, revert").
- **Go/No-Go:** Go if the chosen workflow had ≥ 95% unedited approvals over 4 weeks.
- **Time:** 4 h.

### Week 12 — Documentation and a trained backup approver
- **Goal:** The business does not depend on one person's phone.
- **Tasks:**
  1. Backup approver: set up Telegram (Section 5), then move approvals to a group chat with both of you.
  2. Walk the backup through 3 live approvals with the owner watching.
  3. Print Section 11 (checklist) and Section 12 (troubleshooting) and keep them in the office.
  4. Export every workflow: n8n → each workflow → ⋯ → Download. Save the files to Google Drive `AIOS/backups/2026-W12/`.
  5. Do the monthly Airtable CSV export (Section 7, item 5).
- **Deliverable:** A backup approver who has approved alone at least once, and a backup folder.
- **Go/No-Go:** Program complete if the owner can be offline for 48 h without anything breaking.
- **Time:** 4 h.

**Total:** about 60 hours over 12 weeks, or about 5 h a week.
