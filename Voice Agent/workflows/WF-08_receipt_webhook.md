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

STEP 2 — Find Contact
   Lookup by: customer_phone (from webhook payload)

STEP 3 — Update Contact Field
   Field: order_details
   Value: {{receipt_url}}   ← mapped from webhook payload
```

---

## Notes

- `order_details` is used in WF-01 (customer SMS, admin SMS) and WF-06b (admin alert) as `{{contact.order_details}}`.
- This workflow runs independently — it does not wait for WF-00 or WF-01 to complete.
- If the contact is not found by phone, the field update silently fails — monitor for this during testing.

---

*End of WF-08 — The Greek Grill AB*
