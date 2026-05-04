# WF-00 — Master Router
> The Greek Grill AB | GHL Workflow

---

**Trigger:** Call Ended (GHL native voice trigger)
**Runs:** After every call

---

## Purpose

The single entry point for all post-call automation. Routes the call to the correct sub-workflow based on `last_order_status`. The guard clause (Step 1) is what makes the entire system reliable — it blocks junk calls from polluting the CRM.

---

## Steps

```
STEP 1 — GUARD (most important step)
Check: Is {{contact.last_order_status}} empty or null?
   ├── YES → Apply tag: incomplete-call → EXIT workflow immediately
   │         (hang-up, dead air, bot confusion — do nothing)
   └── NO  → continue to Step 2

STEP 2 — Customer type (only runs if status is populated)
Check: Did this contact exist before this call?
   ├── New contact  → Apply tag: new-customer
   └── Existing     → Apply tag: returning-customer

STEP 3 — Route by outcome
Branch on {{contact.last_order_status}}:
   ├── "Order Confirmed"    → Apply tag: order-confirmed    → WF-01
   ├── "Reservation Booked" → Apply tag: reservation-booked → WF-02
   ├── "Info Only"          → Apply tag: info-only          → WF-03
   └── "Transferred"        → Apply tag: transferred        → WF-04
```

---

## Rules

- **Routing only** — never add SMS, emails, or opportunity creation here. If a new outcome is needed, add a new tag + a new workflow, and update Step 3 only.
- The guard in Step 1 is what stops hang-ups and bot confusion from creating phantom CRM entries.
- WF-05 and WF-06 are spawned from WF-01, not from here.
- WF-07 (missed call) runs on its own separate trigger — completely independent of this workflow.
- WF-08 (receipt webhook) is triggered by n8n directly — completely independent of this workflow.

---

*End of WF-00 — The Greek Grill AB*
