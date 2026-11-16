# AI Three-Way Invoice Matching & Fraud Flagging Agent (n8n): n8n Workflow Code

Importable n8n workflow (n8n Cloud). Copy the JSON below into a file named `invoice-fraud-aiagent-cloud.json`, then in n8n use **Workflows → Import from File** (or paste it onto the canvas with Ctrl+V).

After import, connect your own credentials (no secrets are stored in this file) and follow the setup steps in the README.

````json
{
  "name": "AI powered three-way invoice matching & fraud flagging engine (AI Agent, n8n Cloud)",
  "nodes": [
    {
      "parameters": {
        "content": "## Setup (n8n Cloud)\n1. Credentials (free tiers): Google Gemini, Slack, Postgres (Supabase)\n2. Create table invoice_checks in Supabase SQL Editor\n3. Slack channel: invoice-fraud (change in the Slack node)\n4. Click Execute workflow: 4 mock invoices (clean, price jump, over-billed, risky new vendor)\nThree-way match: purchase order vs goods received vs invoice\nReal client use: replace Mock Documents with a Webhook and your ERP data",
        "height": 260,
        "width": 580
      },
      "id": "4e5eb4f5-03b4-4c11-89ab-9a1df693506d",
      "name": "Setup notes",
      "type": "n8n-nodes-base.stickyNote",
      "typeVersion": 1,
      "position": [
        -60,
        -220
      ]
    },
    {
      "parameters": {},
      "id": "2c4e6541-24a2-42ab-90f4-9543d1bb85af",
      "name": "When clicking Execute workflow",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [
        0,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "return [\n  { json: { invoice_number: 'INV-9001', vendor: 'Brightline Supplies', po_qty: 100, po_price: 5, received_qty: 100, invoice_qty: 100, invoice_price: 5 } },\n  { json: { invoice_number: 'INV-9002', vendor: 'Apex Hosting', po_qty: 10, po_price: 1000, received_qty: 10, invoice_qty: 10, invoice_price: 1250 } },\n  { json: { invoice_number: 'INV-9003', vendor: 'Globex Parts', po_qty: 50, po_price: 20, received_qty: 30, invoice_qty: 50, invoice_price: 20 } },\n  { json: { invoice_number: 'INV-9004', vendor: 'Quick Cash Consulting', po_qty: 1, po_price: 4900, received_qty: 1, invoice_qty: 1, invoice_price: 4900 } }\n];"
      },
      "id": "a03e8882-089f-4eb5-b587-00c98047b248",
      "name": "Mock Documents",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        220,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "return $input.all().map((it) => {\n  const d = it.json;\n  const issues = [];\n  if (d.invoice_qty > d.received_qty) issues.push('billed ' + d.invoice_qty + ' but only ' + d.received_qty + ' received');\n  else if (d.invoice_qty !== d.po_qty) issues.push('quantity differs from the purchase order');\n  const variance = Math.round((d.invoice_price - d.po_price) / d.po_price * 1000) / 10;\n  if (Math.abs(variance) > 5) issues.push('unit price ' + variance + '% off the purchase order');\n  return { json: Object.assign({}, d, {\n    po_amount: d.po_qty * d.po_price,\n    invoice_amount: d.invoice_qty * d.invoice_price,\n    match_result: issues.length ? 'mismatch' : 'matched',\n    match_issues: issues.join('; ') || 'none'\n  }) };\n});"
      },
      "id": "d4fb1af9-401d-46a0-8bbb-7b866bcbf5c5",
      "name": "Match Check",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        440,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Review this invoice for fraud risk.\nInvoice: {{ $json.invoice_number }} from {{ $json.vendor }}\nPO amount: {{ $json.po_amount }} | Invoice amount: {{ $json.invoice_amount }}\nThree-way match: {{ $json.match_result }} ({{ $json.match_issues }})",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are an accounts-payable fraud analyst. Use the vendor_history tool with the vendor name to check the vendor's record. Treat a recent bank-account change or a first invoice from a new vendor as warning signs. Respond with ONLY a JSON object, no markdown, no extra text: {\"fraud_risk\":\"low|medium|high\",\"reason\":\"one sentence\"}"
        }
      },
      "id": "c494d4de-d8f2-4907-8408-47b810459f93",
      "name": "AI Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        680,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "modelName": "models/gemini-2.5-flash",
        "options": {}
      },
      "id": "5a620d00-b82c-460f-88d2-3b9ab52a94ac",
      "name": "Google Gemini Chat Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1,
      "position": [
        660,
        420
      ],
      "credentials": {
        "googlePalmApi": {
          "id": "REPLACE_GEMINI",
          "name": "Google Gemini(PaLM) Api account"
        }
      }
    },
    {
      "parameters": {
        "name": "vendor_history",
        "description": "Returns a vendor's record. Input: the vendor name only, for example 'Apex Hosting'. Returns years as vendor, average invoice, days since the bank account changed and whether this is the first invoice, as JSON.",
        "jsCode": "const vendors = {\n  'brightline supplies': { years_as_vendor: 6, avg_invoice: 520, bank_account_changed_days_ago: 900, first_invoice: false },\n  'apex hosting': { years_as_vendor: 3, avg_invoice: 10000, bank_account_changed_days_ago: 400, first_invoice: false },\n  'globex parts': { years_as_vendor: 4, avg_invoice: 1000, bank_account_changed_days_ago: 300, first_invoice: false },\n  'quick cash consulting': { years_as_vendor: 0, avg_invoice: 0, bank_account_changed_days_ago: 2, first_invoice: true }\n};\nreturn JSON.stringify(vendors[String(query).trim().toLowerCase()] || { error: 'vendor not found' });"
      },
      "id": "94ba5b11-2858-4974-9375-b39eea866345",
      "name": "Vendor History Tool",
      "type": "@n8n/n8n-nodes-langchain.toolCode",
      "typeVersion": 1.3,
      "position": [
        880,
        420
      ]
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Match Check').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst risk = ['low', 'medium', 'high'].includes(String(r.fraud_risk).toLowerCase()) ? String(r.fraud_risk).toLowerCase() : 'medium';\nconst escalate = ctx.match_result === 'mismatch' || risk !== 'low';\nconst parts = [];\nif (ctx.match_result === 'mismatch') parts.push(ctx.match_issues);\nif (r.reason) parts.push(String(r.reason));\nreturn { json: {\n  invoice_number: ctx.invoice_number,\n  vendor: ctx.vendor,\n  po_amount: ctx.po_amount,\n  invoice_amount: ctx.invoice_amount,\n  match_result: ctx.match_result,\n  fraud_risk: risk,\n  decision: escalate ? 'escalate' : 'auto_approve',\n  reason: parts.join('. ').slice(0, 500) || 'three-way match passed, low fraud risk'\n}};"
      },
      "id": "937de254-9dcc-4f9c-bc96-805de2184e69",
      "name": "Parse Decision",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1040,
        200
      ]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "ace20bd4-7f6b-49c6-b4cf-e7685ddff5df",
              "leftValue": "={{ $json.decision }}",
              "rightValue": "escalate",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "279e5c80-9f85-452b-9867-09c8514a34fd",
      "name": "Escalate?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        1260,
        200
      ]
    },
    {
      "parameters": {
        "select": "channel",
        "channelId": {
          "__rl": true,
          "value": "invoice-fraud",
          "mode": "name"
        },
        "text": "=:mag: *Invoice held: {{ $json.invoice_number }}* ({{ $json.vendor }})\nPO: {{ $json.po_amount }} | Invoice: {{ $json.invoice_amount }}\nMatch: {{ $json.match_result }} | Fraud risk: {{ $json.fraud_risk }}\nWhy: {{ $json.reason }}",
        "otherOptions": {}
      },
      "id": "7d9e671d-04d8-491c-a6c7-caf045906096",
      "name": "Slack Alert",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2.3,
      "position": [
        1520,
        120
      ],
      "credentials": {
        "slackApi": {
          "id": "REPLACE_SLACK",
          "name": "Slack account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "operation": "insert",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "invoice_checks",
          "mode": "name"
        },
        "options": {},
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "invoice_number": "={{ $json.invoice_number }}",
            "vendor": "={{ $json.vendor }}",
            "po_amount": "={{ $json.po_amount }}",
            "invoice_amount": "={{ $json.invoice_amount }}",
            "match_result": "={{ $json.match_result }}",
            "fraud_risk": "={{ $json.fraud_risk }}",
            "decision": "={{ $json.decision }}",
            "reason": "={{ $json.reason }}"
          },
          "matchingColumns": [],
          "schema": [
            {
              "id": "invoice_number",
              "displayName": "invoice_number",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "vendor",
              "displayName": "vendor",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "po_amount",
              "displayName": "po_amount",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "invoice_amount",
              "displayName": "invoice_amount",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "match_result",
              "displayName": "match_result",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "fraud_risk",
              "displayName": "fraud_risk",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "decision",
              "displayName": "decision",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "reason",
              "displayName": "reason",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            }
          ]
        }
      },
      "id": "aa43e3b0-8828-4252-8da6-c101cd6549fc",
      "name": "Save Check",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        1520,
        320
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      }
    }
  ],
  "connections": {
    "When clicking Execute workflow": {
      "main": [
        [
          {
            "node": "Mock Documents",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Mock Documents": {
      "main": [
        [
          {
            "node": "Match Check",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Match Check": {
      "main": [
        [
          {
            "node": "AI Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "AI Agent": {
      "main": [
        [
          {
            "node": "Parse Decision",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Decision": {
      "main": [
        [
          {
            "node": "Escalate?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Escalate?": {
      "main": [
        [
          {
            "node": "Slack Alert",
            "type": "main",
            "index": 0
          },
          {
            "node": "Save Check",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Save Check",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Google Gemini Chat Model": {
      "ai_languageModel": [
        [
          {
            "node": "AI Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Vendor History Tool": {
      "ai_tool": [
        [
          {
            "node": "AI Agent",
            "type": "ai_tool",
            "index": 0
          }
        ]
      ]
    }
  },
  "pinData": {},
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "tags": []
}
````
