# Multi-Channel Inventory Sync — Prototype

Event-driven inventory sync service built as a mini-prototype in **n8n*.
Prevents overselling when the same SKU is sold across multiple e-commerce channels.

## Core value

Keeps stock counts consistent across **Shopify, Kilimall, and Jumia**. When an order webhook fires on one channel, the service
checks and decrements a central inventory ledger, then pushes the updated
stock count to all other channels in near real time — blocking a second
sale of the same last unit instead of letting it oversell.

## How it works

1. **Webhook receiver** — accepts an incoming "order placed" event from a channel.
2. **Duplicate check** — compares the event ID against the ledger; repeated/retried events are ignored, preventing double-decrements.
3. **Stock check** — confirms enough stock is available before confirming the sale.
4. **Ledger decrement** — on a valid sale, the central ledger (Google Sheets) is updated.
5. **Fan-out** — the new stock count is pushed in parallel to the other four channels.
6. **Response** — the service returns one of three JSON statuses: `confirmed`, `rejected` (out of stock), or `duplicate`.

## Prototype stack

| Piece | Tool used (prototype) | Production equivalent |
| Workflow / logic | n8n | n8n or custom backend service |
| Inventory ledger | Google Sheets | PostgreSQL / real database |
| Channel endpoints | webhook.site (mocked) | Real Shopify,Kilimall, Jumia APIs |
| Event simulation | Postman | Real channel webhooks |

## Contents of this repo

- `workflow.json` — exported n8n workflow (import into n8n to view/run)
- Learning and blocker journal


## Demonstrated outcomes

- ✅ Valid new order → stock decremented, pushed to all channels, `confirmed` returned
- ✅ Competing sale for the last unit → correctly `rejected` instead of oversold
- ✅ Retried/duplicate event → ignored, returns `duplicate`, no double-decrement

## Notes

This is a prototype built to prove the core concept, not a production
system. Channel integrations are mocked, and Google Sheets.


Sample payloads

1. Valid new order (expect confirmed)

json
{
  "sku": "SKU-001",
  "quantity": 1,
  "id": "evt-demo-A",
  "source": "shopify"
}

Expected response:

json
{
  "status": "confirmed",
  "message": "Stock updated successfully"
}

2. Competing sale for the same SKU when stock is now insufficient (expect rejected)

json
{
  "sku": "SKU-001",
  "quantity": 1,
  "id": "evt-demo-B",
  "source": "Jumia"
}

Expected response:

json
{
  "status": "rejected",
  "reason": "out of stock"
}

3. Resending the same event ID as #1 (expect duplicate)

json
{
  "sku": "SKU-001",
  "quantity": 1,
  "id": "evt-demo-A",
  "source": "shopify"
}

Expected response:

json
{
  "status": "duplicate",
  "message": "Event already processed, ignored"
}

How to test
Method: POST
URL: https://lasborn.app.n8n.cloud/webhook/order-webhook
Body type: raw / JSON
Tool: Postman

