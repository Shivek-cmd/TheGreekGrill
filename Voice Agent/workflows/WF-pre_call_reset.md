# WF-pre — Pre-Call Reset
> The Greek Grill AB | GHL Workflow

---

**Trigger:** Call Started (GHL native voice trigger)
**Runs:** At the start of every inbound call, before Sofia speaks

---

## Purpose

Clears all per-call custom fields and per-call tags before a new call begins. Without this, leftover data from the previous call can conflict with the new order — wrong items, wrong type, wrong status.

---

## Steps

```
STEP 1 — Clear Custom Fields
   Clear all per-call contact fields:
   - last_order_status
   - last_order_type
   - last_order_items
   - last_delivery_address
   - last_ready_time
   - special_requests
   - buzzer_code
   - last_receipt_url
   - order_details

STEP 2 — Remove Per-Call Tags
   Remove tags:
   - pickup-order
   - delivery-order
   - incomplete-call
   - new-customer
   - returning-customer
   - order-confirmed
   - reservation-booked
   - info-only
   - transferred
```

> **Why:** Prevents the next call from inheriting stale data. Lifetime tags (`vip-customer`, `delivery-customer`, `review-positive`, etc.) are NOT removed — those accumulate intentionally.

---

*End of WF-pre — The Greek Grill AB*
