## Section 14 — Top 3 Mistakes + Fastest First Win

### The single biggest mistake: letting an AI-made invoice reach a customer instead of a ZRA fiscal invoice

It is tempting to switch on "email invoice to customer" in QuickBooks, or to let n8n send a nice PDF, because it feels like the finished job. **It is not a valid tax invoice.** Here is what follows:
- **Your customer cannot claim the input VAT.** Chongwe Builders' accountant rejects the ZMW 17,992.80 VAT on ORD-2026-0147, and you lose a repeat B2B customer.
- **ZRA can disallow the sale records and apply penalties** for sales not issued through Smart Invoice. In an audit, every non-fiscal invoice in your QuickBooks becomes a separate finding.
- **Your books and ZRA's records no longer match.** The Reconciliation Agent has nothing to reconcile against, and by month-end you cannot tell which sales were fiscalised.

**What to do:** keep the W2 design exactly as built. The QuickBooks invoice is internal, the Telegram message tells you to issue the fiscal invoice, and `ZRA_InvoiceNo` must be filled within 48 h (Dashboard tile C9 turns 🔴 otherwise).

### Two more mistakes to avoid
2. **Skipping shadow mode.** Going live with the Sales Agent on day one means the first wrong species or wrong volume goes straight to a customer. One week of drafts that you check costs 5 hours and catches most prompt errors before anyone sees them.
3. **Letting the AI quote prices.** Prices depend on log costs, kiln time and the customer relationship, and the model doesn't know any of these. A model that tells a contractor "Rosewood is USD 900/m³" when your price is USD 1,450 has just cost you USD 2,200 on a 4 m³ order. The prompt forbids prices, and TestCases checks that none appear.

### The fastest first win: Sales Agent in shadow mode in 3 days

**Day 1 (about 3 h):**
1. Sign up at **n8n.io → Get started (Cloud)**. Settings → timezone **Africa/Lusaka**.
2. **platform.openai.com** → API keys → Create, then Settings → Limits → USD 25 cap.
3. Airtable → create base `AIOS_OPS` → **+ Add or import → CSV** → `templates/airtable-Leads.csv`. Set `TPIN` to Single line text and `Species` to Single select.
4. Telegram → @BotFather → `/newbot` → token. Then @userinfobot → your chat ID. Open your bot and press **Start**.

**Day 2 (about 2 h):**
5. Gmail → create the label **Inquiries** + a filter (Section 4.3, W1 step 1).
6. n8n → Credentials: Gmail, Google Sheets, Airtable, OpenAI, Telegram.
7. n8n → Create Workflow → paste `n8n/W1-AIOS-Sales-Lead-Intake.json`. Click each ⚠️ node and pick its credential, base, table or sheet. Config → your chat ID. Save.
8. **Test workflow** → send yourself test email **TC01** (Section 9) → check: Airtable lead created ✔, Gmail draft ✔, Telegram 🪵 ✔.

**Day 3 (about 1 h):**
9. Send TC07 (escalation), TC16 (spam) and TC19 (injection). Expected: ⚠️ with no draft, no lead, and ⚠️ with no draft.
10. Toggle W1 **Active**. From now on every real enquiry has a lead and a draft waiting within about 2 minutes. You keep replying yourself and compare.

You have now saved the time spent typing lead details into a spreadsheet, and every enquiry is logged, with nothing sent that you haven't seen.

### Do this today
**Submit the VAT registration to ZRA this week, and ask your accountant one question: "How should our QuickBooks invoices become ZRA Smart Invoice fiscal invoices: through the portal or an accredited connector?"** Live invoicing (Week 6) cannot start until the VAT certificate has arrived and that question is answered. Starting the registration now means it doesn't hold up the rollout.
