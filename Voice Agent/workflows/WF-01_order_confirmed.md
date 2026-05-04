# WF-01 — Order Confirmed
> The Greek Grill AB | GHL Workflow

---

**Trigger:** Tag added: `order-confirmed`
**Spawns:** WF-05 (re-engagement), WF-06 (Google review)

---

## Steps

```
STEP 1 — Apply order sub-tag
   ├── last_order_type = "pickup"   → Tag: pickup-order
   └── last_order_type = "delivery" → Tag: delivery-order

STEP 2 — Check special_requests
   └── Not empty → Tag: has-special-request

STEP 3 — Create Opportunity
   Pipeline: Restaurant Orders
   Stage:    New Order
   Name:     {{contact.name}} — {{contact.last_order_type}} — {{contact.last_call_date}}
   Owner:    Admin

STEP 4 — Send SMS to Admin (immediate)
   ─────────────────────────────────────────────────
   🍽️ NEW ORDER — The Greek Grill
   Name:     {{contact.name}}
   Phone:    {{contact.phone}}
   Type:     {{contact.last_order_type}}
   Receipt:  {{contact.order_details}}
   Ready By: {{contact.last_ready_time}}
   [If delivery] Address: {{contact.last_delivery_address}}
   [If delivery] Buzzer:  {{contact.buzzer_code}}
   [If allergy] ⚠️ Special: {{contact.special_requests}}
   ─────────────────────────────────────────────────

STEP 5 — Send Email to Admin
   Subject: New Order — {{contact.name}} — {{contact.last_ready_time}}
   Body: Full order detail (same fields as SMS, formatted)

STEP 6 — Send SMS to Customer
   ─────────────────────────────────────────────────
   Hi {{contact.name}}! Your order at The Greek Grill is confirmed.
   Ready by {{contact.last_ready_time}}.
   View your receipt: {{contact.order_details}}
   📍 #31 99 Wye Rd, Sherwood Park — see you soon!
   ─────────────────────────────────────────────────

STEP 7 — VIP Check
   Count confirmed orders for this contact
   └── 5+ orders → Apply tag: vip-customer

STEP 8 — Enroll in WF-05
   Re-engagement timer starts now

STEP 9 — Apply tag: review-pending
   → Triggers WF-06 (review request after 3 hours)
```

---

## Notes

- `{{contact.order_details}}` is the clickable PDF receipt URL — written by WF-08 when n8n fires Node 6.
- Admin SMS is always immediate — no wait.
- Customer SMS is sent right after admin SMS — no wait.

---

*End of WF-01 — The Greek Grill AB*
