# WF-08 — Receipt Webhook (Inbound from n8n)
> The Greek Grill AB | GHL Workflow

---

**Trigger:** Inbound Webhook (GHL native — fired by n8n Node 6)
**Runs:** Right after the PDF receipt is generated, in parallel with the call ending
**Does NOT depend on WF-00** — triggered directly by n8n, not by a tag

---

## Purpose

Receives the receipt URL from n8n after PDF generation → finds the contact by phone number → writes the URL to `order_details`. Once this runs, `{{contact.order_details}}` is available in all SMS and email templates as a clickable receipt link.

---

## Webhook Details

**GHL Inbound Webhook URL:**
```
https://services.leadconnectorhq.com/hooks/EXi2BTQg5OR7dUmjnc3N/webhook-trigger/4b0c4995-0497-4811-ada3-e10d8041d44b
```

**Payload received from n8n Node 6:**
```json
{
  "receipt_url":    "https://...",
  "order_id":       "1777470908216",
  "customer_phone": "+919413752688"
}
```

---

## Steps

```
STEP 1 — Inbound Webhook fires
   (n8n Node 6 POSTs after PDF is generated)
   Payload: { receipt_url, order_id, customer_phone }

STEP 2 — Find Contact
   Lookup by: customer_phone → {{inboundWebhookRequest.customer_phone}}

STEP 3 — Update Contact Field
   Field: order_details
   Value: {{inboundWebhookRequest.receipt_url}}

STEP 4 — Find Opportunity  ← REQUIRED before Update Opportunity
   Action:    Find Opportunity
   Label:     Find Latest Order Opportunity
   Retrieve:  Latest
   Filters (AND):
     - Pipeline = Restaurant Orders
     - Status   = Open

   ├── Opportunity Found  → continue to STEP 5
   └── Opportunity Not Found → EXIT (receipt still saved on contact in Step 3)

STEP 5 — Update Opportunity
   Action:  Update Opportunity
   Fields:
     - Order_details: {{inboundWebhookRequest.receipt_url}}
   Allow Move to Previous Stage: OFF
   Allow Duplicate: OFF
```

---

## Notes

- `order_details` is used in WF-01 (customer SMS, admin SMS) and WF-06b (admin alert) as `{{contact.order_details}}`.
- This workflow runs independently — it does not wait for WF-00 or WF-01 to complete.
- If the contact is not found by phone, the field update silently fails — monitor for this during testing.

---

*End of WF-08 — The Greek Grill AB*
