# AI Three-Way Invoice Matching & Fraud Flagging Agent (n8n)

An n8n workflow that matches each invoice against its purchase order and goods receipt with plain rules, then lets an AI Agent (Gemini 3.5 Flash Lite) judge fraud risk using the vendor's history. Mismatches and risky vendors are escalated to Slack, clean invoices are approved automatically.

▶ **Demo video:** https://drive.google.com/file/d/13azotrmxnFggLKOcJz43WbeO_sHuItMo/view

## How it works

1. Mock documents arrive (purchase order quantity and price, received quantity, invoiced quantity and price)
2. A Code node runs the three-way match: billed more than received, quantity differs from the PO, or unit price more than 5% off the PO is a mismatch
3. The AI Agent calls the `vendor_history` tool (years as vendor, average invoice, days since bank account change, first invoice) and returns fraud risk and a reason as JSON
4. A Code node validates it. A mismatch or any risk above low means `escalate`, otherwise `auto_approve`
5. Escalated invoices go to Slack, every check is saved to Postgres

```
Manual Trigger -> Mock Documents -> Match Check -> AI Agent (Gemini + vendor_history tool)
-> Parse Decision -> Escalate?
   yes -> Slack Alert -> Save Check
   no  -> Save Check
```

## AI Agent

AI Agent node with one tool (`vendor_history`). The match result is code, so a mismatch is always escalated no matter what the AI says.

## Tech stack

n8n, AI Agent, Gemini 3.5 Flash Lite, Slack, Postgres (Supabase)

## Setup

1. Copy the JSON from `workflow.md` into a file named `invoice-fraud-aiagent-cloud.json` and import it into n8n
2. Create credentials (all have free tiers):
   - Google Gemini (API key)
   - Slack (Bot token and signing secret)
   - Postgres (Supabase Session pooler, SSL: Require)
3. In the Slack node set channel `invoice-fraud`, then run `/invite @your-bot` in that channel
4. Create the database tables (below)
5. In the Postgres save node remove the `id` and `created_at` fields, the database fills them
6. Click **Execute workflow**: the mock inputs run through all branches

No other API keys are needed, the tool nodes are self-contained mock tools.

## Database

```sql
create table if not exists public.invoice_checks (
  id bigint generated always as identity primary key,
  invoice_number text,
  vendor text,
  po_amount numeric,
  invoice_amount numeric,
  match_result text,
  fraud_risk text,
  decision text,
  reason text,
  created_at timestamptz default now()
);
```

## Demo data

| Mock input | Expected result |
|---|---|
| INV-9001 Brightline Supplies: everything matches | Auto-approve if vendor history looks normal |
| INV-9002 Apex Hosting: unit price 25% above the PO | Escalate (price mismatch) |
| INV-9003 Globex Parts: billed 50 but only 30 received | Escalate (quantity mismatch) |
| INV-9004 Quick Cash Consulting: amounts match | Depends on vendor_history, escalated if it shows warning signs |

## Notes

- Replace Mock Documents with PO, receipt and invoice data from your ERP, and `vendor_history` with your vendor master for client use
- The 5% price tolerance is an example, change it in the Match Check node
- Slack message text must be in **Expression** mode so `{{ }}` values are filled in
- Never commit API keys. Use n8n credentials only

## Author

Zarin Tasnim Shaily, AI Automation Engineer
[LinkedIn](https://linkedin.com/in/5ha1l72) | [GitHub](https://github.com/ztsprofessionalaiid7-alt)
