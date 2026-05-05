# WF-01 — Order Confirmed
> The Greek Grill AB | GHL Workflow

---

**Trigger:** Added to Workflow by WF-00 (Add to Workflow action — no tag trigger)
**Spawns:** WF-05 (re-engagement), WF-06 (Google review)

---

## Steps

```
STEP 0 — Clean up re-engagement tags (if customer was inactive before this order)
   Remove tag: inactive-10-days   (if exists — customer is active again)
   Remove tag: inactive-17-days   (if exists — customer is active again)

STEP 1 — Apply order sub-tags
   ├── last_order_type = "pickup"   → Tag: pickup-order
   └── last_order_type = "delivery" → Tag: delivery-order

STEP 2 — Check special_requests
   └── Not empty → Tag: has-special-request

STEP 3 — Create Opportunity
   Action:   Create Opportunity  (NOT "Create or Update" — deprecated)
   Pipeline: Restaurant Orders
   Stage:    New Order
   Name:     {{contact.name}} — {{contact.last_order_type}} — {{contact.last_call_date}}
   Status:   Open
   Allow Duplicate Opportunities: ON  ← each order = 1 new opportunity, intentional

STEP 4 — Find Opportunity
   Action:            Find Opportunity
   Label:             Find Latest Order Opportunity
   Retrieve:          Latest (most recently created)
   Filters (AND):
     - Pipeline       = Restaurant Orders
     - Status         = Open
     - Opportunity Name contains {{contact.name}}

   ├── Opportunity Found →
   │     The just-created opportunity is now available as a reference.
   │     Use {{opportunity.id}}, {{opportunity.name}}, etc. in subsequent steps.
   │     Continue to STEP 5.
   │
   └── Opportunity Not Found →
         Should not happen (we just created one).
         Safety: Log / notify admin, then continue to STEP 5.

STEP 5 — Increment order_count
   Action:             Math Operation
   Select Field:       order_count
   Operator:           Add
   Value:              1
   Save Result To:     order_count

STEP 6 — Branch: Customer milestone
   ├── order_count = 1
   │     Apply tag:  new-customer        (first order ever — lifetime marker)
   │     (no removal needed — they have no prior tags to clean)
   │
   └── order_count > 1
         Remove tag: new-customer        (they are no longer new — prevent new-customer campaigns targeting them)
         Apply tag:  returning-customer  (per-call status — WF-pre removes it before next call)

   Separately (not exclusive with above):
   └── order_count >= 5
         Apply tag: vip-customer         (lifetime milestone — additive, no other tags removed)
         (returning-customer stays — VIP customers are still returning customers)

STEP 7 — Increment delivery_count (only if delivery order)
   Condition: last_order_type = "delivery"
   Action:             Math Operation
   Select Field:       delivery_count
   Operator:           Add
   Value:              1
   Save Result To:     delivery_count

   └── delivery_count = 2 → Tag: delivery-customer  (lifetime tag — 2nd delivery reached)

STEP 8 — Send SMS to Admin (immediate — no wait)
   ─────────────────────────────────────────────────
   🍽️ NEW ORDER — The Greek Grill
   Name:     {{contact.name}}
   Phone:    {{contact.phone}}
   Type:     {{contact.last_order_type}}
   Receipt:  {{contact.order_details}}  (contains receipt link)
   Ready By: {{contact.last_ready_time}}
   [If delivery] Address: {{contact.last_delivery_address}}
   [If delivery] Buzzer:  {{contact.buzzer_code}}
   [If allergy] ⚠️ Special: {{contact.special_requests}}
   ─────────────────────────────────────────────────

STEP 9 — Send Email to Admin
   Subject: New Order — {{contact.name}} — {{contact.last_ready_time}}
   Body: Full order detail (same fields as SMS, formatted)

STEP 10 — Send SMS to Customer
   ─────────────────────────────────────────────────
   Hi {{contact.name}}! Your order at The Greek Grill is confirmed.
   Ready by {{contact.last_ready_time}}.
   View your receipt: {{contact.order_details}}
   📍 #31 99 Wye Rd, Sherwood Park — see you soon!
   ─────────────────────────────────────────────────

STEP 11 — Enroll in WF-05
   Re-engagement timer starts now (or resets if already enrolled from prior order)

STEP 12 — Apply tag: review-pending
   → Triggers WF-06 (review request after 3 hours)
   → WF-06 owns removing this tag after use
   → If review-requested tag already exists: WF-06 will guard and exit — no double SMS
```

---

## Notes

- `order_count` and `delivery_count` are **never cleared** by WF-pre — they are lifetime counters. Only add to them, never reset.
- `{{contact.order_details}}` is the clickable PDF receipt URL written by WF-08 when n8n fires Node 6. It may arrive slightly after the SMS — WF-08 runs in parallel with WF-00/WF-01.
- Admin SMS is always immediate — never add a wait before it.
- `new-customer` is a lifetime tag — applied once at order_count = 1, never removed.
- `vip-customer` is a lifetime milestone — once applied at order_count = 5, subsequent orders don't remove it.

---

*End of WF-01 — The Greek Grill AB*
