# AIOS for a Zambian Hardwood Manufacturer

A complete, no-code build package for an **AI Operating System (AIOS)** that runs a hardwood timber business (Rosewood, Teak, Mahogany, Kiaat) from supplier sourcing to payments. It uses n8n, Airtable, Google Sheets, Gmail, QuickBooks Online and a Telegram approval bot. A human approves every money-moving step, and every sale is invoiced through ZRA Smart Invoice.

Built for **Abijeff Investments Limited** (Lusaka; registered with PACRA). Customer and supplier names, buyer TPINs and QuickBooks IDs in the examples are invented samples. Replace them with real ones as you configure each tool.

## Start here
1. Read **[Section 14 — Fastest first win](docs/14-mistakes-first-win.md)** (5 minutes) and do the "do this today" step.
2. Paste **[`AIOS-Build-Bible.md`](AIOS-Build-Bible.md)** (all 14 sections in one file) into a Google Doc titled "AIOS Build Bible".
3. Follow the **[90-day plan](docs/08-rollout-plan.md)** week by week, ticking off the **[printable checklist](docs/11-build-checklist.md)**.
4. Send Sections 3, 4 and 7 to your accountant before Week 5.

## What's in the package

| Folder / file | What it is |
|---|---|
| `AIOS-Build-Bible.md` | All 14 sections combined, for pasting into Google Docs |
| `docs/01-architecture.md` … `docs/14-mistakes-first-win.md` | The same sections as separate files |
| `n8n/W0…W9 *.json` | **Importable n8n workflows**, one per agent plus support: Error Handler, Sales Lead Intake, Invoice Preparation, Inventory Watchdog, Sourcing Email Parser, Orders/Leads → Sheets sync, Weekly Report, Procurement, Payments Matching, Weekly Reconciliation |
| `prompts/*.md` | The 8 agent system prompts (copy into n8n AI Agent → System Message, and into the Airtable Prompts table) |
| `templates/airtable-*.csv` | Import files for the 5 Airtable tables, with sample rows |
| `templates/sheets-*.csv` | Import files for the Inventory, AuditLog, TokenCosts and TestCases tabs |
| `tools/build_workflows.py` | Regenerates the n8n JSON, Section 3 and the Build Bible from `prompts/`. Optional, for a technical helper |

## Sections
1. [Architecture diagram](docs/01-architecture.md)
2. [Data schemas](docs/02-data-schemas.md)
3. [Agent specifications (8 agents, full prompts)](docs/03-agent-specs.md)
4. [n8n workflows: import and setup](docs/04-n8n-workflows.md)
5. [Human-in-the-loop: Telegram approvals](docs/05-hitl-telegram.md)
6. [Idempotency and error handling](docs/06-idempotency-errors.md)
7. [ZRA compliance checklist](docs/07-zra-compliance.md)
8. [90-day rollout plan](docs/08-rollout-plan.md)
9. [Test eval set: 20 emails](docs/09-test-eval-set.md)
10. [KPI dashboard spec](docs/10-kpi-dashboard.md)
11. [Build order checklist](docs/11-build-checklist.md)
12. [Troubleshooting: top 15](docs/12-troubleshooting.md)
13. [Cost breakdown: about $1,200–1,500 in Year 1](docs/13-cost-breakdown.md)
14. [Top mistakes and fastest first win](docs/14-mistakes-first-win.md)

## How the workflow JSON was checked
- Every node type, version, parameter name and option value was checked against the official n8n node definitions (`n8n-nodes-base` 2.15.1 and `@n8n/n8n-nodes-langchain` 2.48.5).
- The JavaScript in every Code node (invoice maths, stock flags, PO pack, payment allocation, reconciliation exceptions, report pack) was run against sample data.
- The workflows have **not** been run end-to-end against live Gmail, Airtable or QuickBooks accounts, because that needs your credentials. Section 8, Weeks 2–6, is the test plan for that, starting with the QuickBooks Sandbox.
