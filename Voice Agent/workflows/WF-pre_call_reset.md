# WF-pre — Pre-Call Reset
> The Greek Grill AB | GHL Workflow

---

**Trigger:** Call Started (GHL native voice trigger)
**Runs:** At the start of every inbound call, before Sofia speaks

---

## Purpose

Clears all per-call custom fields and per-call tags before a new call begins. Without this, leftover data from the previous call conflicts with the new order — stale tags re-trigger workflows, wrong items/type/status bleed into the next call.

---

## Steps

```
STEP 1 — Clear Per-Call Custom Fields
   Clear (set to empty):
   - last_order_status
   - last_order_type
   - last_order_items
   - last_delivery_address
   - last_ready_time
   - special_requests
   - buzzer_code
   - last_receipt_url
   - order_details

   Do NOT clear (lifetime fields):
   - order_count          ← lifetime order counter
   - delivery_count       ← lifetime delivery counter
   - preferred_language   ← save behaviour protects this anyway

STEP 2 — Remove Per-Call Tags
   Remove:
   - order-confirmed       ← re-applied by WF-00 if needed next call
   - reservation-booked    ← re-applied by WF-00 if needed next call
   - info-only             ← re-applied by WF-00 if needed next call
   - transferred           ← re-applied by WF-00 if needed next call
   - incomplete-call       ← re-applied by WF-00 guard if needed next call
   - missed-call           ← re-applied by WF-07 if needed; clear so it doesn't linger
   - pickup-order          ← per-call order sub-tag
   - delivery-order        ← per-call order sub-tag
   - has-special-request   ← per-call kitchen flag
   - returning-customer    ← per-call status; re-applied by WF-01 if order_count > 1
   - review-pending        ← safety net: if WF-06 didn't clean it up, clear it now

   Do NOT remove (lifetime tags):
   - new-customer          ← permanent first-order marker
   - vip-customer          ← permanent milestone
   - delivery-customer     ← permanent milestone
   - review-requested      ← permanent guard — prevents double review asks
   - review-positive       ← permanent history
   - review-negative       ← permanent history
   - inactive-10-days      ← cleaned up by WF-01 when customer re-orders, not here
   - inactive-17-days      ← cleaned up by WF-01 when customer re-orders, not here
```

---

## Why These Specific Decisions

- `review-pending` is removed here as a safety net only. WF-06 is the primary owner — it removes `review-pending` after the SMS is sent. WF-pre removing it only catches the case where a customer calls back before WF-06 fires.
- `new-customer` is NOT removed — it's a lifetime historical tag. Once a contact places their first order, that fact is permanent.
- `inactive-10-days` / `inactive-17-days` are NOT removed here — they are cleaned up by WF-01 at the top of the next order flow, which is the correct place (active customer context).

---

*End of WF-pre — The Greek Grill AB*
