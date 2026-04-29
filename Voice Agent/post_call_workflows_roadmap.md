# Post-Call Workflows — Roadmap
> The Greek Grill AB | GHL Voice Agent → Post-Call Automation

---

## How This System Works

The system uses **two separate entry points** — not just one. A single "Call Ended" trigger is not enough because it fires for every call including instant hang-ups and missed calls. Using it blindly causes phantom tags, wrong pipeline entries, and meaningless data.

```
┌──────────────────────────────────────────────────────────────┐
│  ENTRY POINT 1: Call Ended (GHL native)                      │
│  → WF-00: MASTER ROUTER                                       │
│    Step 1: Guard — is last_order_status populated?            │
│      ├── NO  → Tag: incomplete-call → EXIT (do nothing)       │
│      └── YES → apply customer type tag, then route            │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  ENTRY POINT 2: Missed Call (GHL native — separate trigger)  │
│  → WF-07: MISSED CALL                                         │
│    Runs independently of WF-00 — never overlaps               │
└──────────────────────────────────────────────────────────────┘
```

**Full routing map (Entry Point 1 only):**

```
CALL ENDS
    │
    ▼
┌─────────────────────────┐
│   WF-00: MASTER ROUTER  │  ← Triggered by: Call Ended
└─────────────────────────┘
    │
    ▼
GUARD: Is last_order_status empty or null?
    ├── YES → Tag: incomplete-call → EXIT immediately
    └── NO  → continue
    │
    ├── Is new contact? ──────────────── Tag: new-customer
    │   OR returning contact? ─────────  Tag: returning-customer
    │
    └── Branch on last_order_status
            │
            ├── "Order Confirmed"    ──→  Tag: order-confirmed  ──→  WF-01
            │                                                          ├──→ WF-05 (re-engagement timer starts)
            │                                                          └──→ WF-06 (review request, 3hr delay)
            ├── "Reservation Booked" ──→  Tag: reservation-booked ─→ WF-02
            ├── "Info Only"          ──→  Tag: info-only ──────────→ WF-03
            └── "Transferred"        ──→  Tag: transferred ────────→ WF-04
```

> WF-05 and WF-06 are NOT triggered by WF-00. They are both spawned from WF-01 after an order is confirmed. WF-07 (missed call) runs on its own trigger — completely separate from WF-00.

---

## Tag Taxonomy

> Tags are permanent labels on the contact. They accumulate over time — a contact can have `new-customer`, `order-confirmed`, `vip-customer` all at once. Design every workflow to be trigger-safe: adding a tag triggers the workflow once only.

### Status Tags (applied per call)
| Tag | Applied When | Purpose |
|-----|-------------|---------|
| `incomplete-call` | Call ended but `last_order_status` is empty/null | Guard tag — stops WF-00 routing, useful for filtering junk calls |
| `missed-call` | GHL missed call trigger fires (WF-07) | Triggers callback SMS |
| `new-customer` | `last_order_status` has a value AND contact is new | Welcome flow — only applied to meaningful calls |
| `returning-customer` | `last_order_status` has a value AND contact existed | Loyalty tracking |
| `order-confirmed` | `last_order_status` = Order Confirmed | Trigger WF-01 |
| `reservation-booked` | `last_order_status` = Reservation Booked | Trigger WF-02 |
| `info-only` | `last_order_status` = Info Only | Trigger WF-03 |
| `transferred` | `last_order_status` = Transferred | Trigger WF-04 |

### Order Sub-Tags (applied inside WF-01)
| Tag | Applied When | Purpose |
|-----|-------------|---------|
| `pickup-order` | `last_order_type` = pickup | Filter/report pickup volume |
| `delivery-order` | `last_order_type` = delivery | Filter/report delivery volume |
| `has-special-request` | `special_requests` is not empty | Kitchen awareness, allergy tracking |
| `review-pending` | End of every WF-01 | Triggers WF-06 (review request after 3 hrs) |

### Review Tags (applied inside WF-06)
| Tag | Applied When | Purpose |
|-----|-------------|---------|
| `review-requested` | Review SMS sent to customer | Track who has been asked |
| `review-positive` | Customer replied HAPPY or 👍 | Potential Google review candidate |
| `review-negative` | Customer replied HELP or 👎 | Alert admin, handle complaint |

### Re-engagement Tags (applied inside WF-05)
| Tag | Applied When | Purpose |
|-----|-------------|---------|
| `inactive-10-days` | No new order for 10 days | Wave 1 win-back SMS sent |
| `inactive-17-days` | No new order for 17 days | Wave 2 win-back SMS sent |

### Lifecycle Tags (applied by milestone logic)
| Tag | Applied When | Purpose |
|-----|-------------|---------|
| `vip-customer` | Contact has 5+ confirmed orders | Priority treatment, future loyalty perks |
| `delivery-customer` | Has placed 2+ delivery orders | Delivery-specific promotions |

---

## Pipelines

Two separate pipelines — one per call outcome type. Keep them separate so order volume and reservation volume are tracked independently.

### Pipeline 1 — Restaurant Orders

| Stage | Who Moves It | When |
|-------|-------------|------|
| **New Order** | Auto (WF-01) | Opportunity created at call end |
| **In Preparation** | Manual (staff) | Kitchen starts working on it |
| **Ready** | Manual (staff) | Order is ready for pickup / dispatched |
| **Completed** | Manual or Auto | Order picked up / delivered |
| **Cancelled** | Manual (staff) | Customer cancelled or no-show |

> Opportunity name format: `[Name] — [Type] — [Date]`
> Example: `Jessica — Delivery — 2026-04-28`
> Opportunity value: pulled from `last_order_items` (text) — monetary value added manually or via future automation

### Pipeline 2 — Table Reservations

| Stage | Who Moves It | When |
|-------|-------------|------|
| **Booked** | Auto (WF-02) | Opportunity created at call end |
| **Reminder Sent** | Auto (WF-02, day before) | Automated reminder triggered |
| **Arrived** | Manual (host) | Party seated |
| **Completed** | Manual (host) | Party finished and left |
| **No Show / Cancelled** | Manual (host) | Didn't arrive or cancelled |

---

## Workflow Blueprints

---
### WF-0(-1) Making All the previous custom fields empty and some tags(VERY VERY IMPORTANT)
**Trigger:** Call Started (GHL native voice trigger)
**Runs:** When user calls

1. Remove all the custom field yo have created 
2. Remove tags
a. pickup-order
b.deliver-order 
c.incomplete-call
d.new-customer
e.returning-customer

Reason: Becuase we dont want to track this for the next call it will confuse right.
Othervise it will conflict with the new orders. Althougth we are storing data in opportunities.


### WF-00 — Master Router
**Trigger:** Call Ended (GHL native voice trigger)
**Runs:** After every call

```
STEP 1 — GUARD (most important step)
Check: Is {{contact.last_order_status}} empty or null?
   ├── YES → Apply tag: incomplete-call → EXIT workflow immediately
   │         (hang-up, dead air, bot confusion — do nothing)
   └── NO  → continue to Step 2

STEP 2 — Customer type (only runs if status is populated)
Check: Did this contact exist before this call?
   ├── New contact → Apply tag: new-customer
   └── Existing contact → Apply tag: returning-customer

STEP 3 — Route by outcome
Branch on {{contact.last_order_status}}:
   ├── "Order Confirmed"    → Apply tag: order-confirmed    → WF-01
   ├── "Reservation Booked" → Apply tag: reservation-booked → WF-02
   ├── "Info Only"          → Apply tag: info-only          → WF-03
   └── "Transferred"        → Apply tag: transferred        → WF-04
```

> The guard in Step 1 is what makes this system reliable. Without it, every hang-up pollutes your CRM with `new-customer` tags and meaningless routing. With it, only calls where Sofia completed at least one action get processed.

---

### WF-01 — Order Confirmed
**Trigger:** Tag added: `order-confirmed`

```
1. Apply order sub-tag
   ├── last_order_type = "pickup"   → Tag: pickup-order
   └── last_order_type = "delivery" → Tag: delivery-order

2. Check special_requests
   └── Not empty → Tag: has-special-request

3. Create Opportunity
   Pipeline: Restaurant Orders
   Stage: New Order
   Name: {{contact.name}} — {{contact.last_order_type}} — {{contact.last_call_date}}
   Owner: Admin

4. Send SMS to Admin
   ─────────────────────────────────────────────────
   🍽️ NEW ORDER — The Greek Grill
   Name: {{contact.name}}
   Phone: {{contact.phone}}
   Type: {{contact.last_order_type}}
   Receipt: {{contact.order_details}}
   Ready By: {{contact.last_ready_time}}
   [If delivery] Address: {{contact.last_delivery_address}}
   [If delivery] Buzzer: {{contact.buzzer_code}}
   [If allergy] ⚠️ Special: {{contact.special_requests}}
   ─────────────────────────────────────────────────

5. Send Email to Admin
   Subject: New Order — {{contact.name}} — {{contact.last_ready_time}}
   Body: Full order detail (same fields as SMS, formatted)

6. Send SMS to Customer
   ─────────────────────────────────────────────────
   Hi {{contact.name}}! Your order at The Greek Grill is confirmed.
   Ready by {{contact.last_ready_time}}.
   View your receipt: {{contact.order_details}}
   📍 #31 99 Wye Rd, Sherwood Park — see you soon!
   ─────────────────────────────────────────────────

7. [VIP Check] Count confirmed orders for this contact
   └── 5+ orders → Apply tag: vip-customer

8. Enroll contact in WF-05 (re-engagement timer — starts now)

9. Apply tag: review-pending  →  triggers WF-06 (review request after 3 hours)
```

---

### WF-02 — Reservation Booked
**Trigger:** Tag added: `reservation-booked`

```
1. Create Opportunity
   Pipeline: Table Reservations
   Stage: Booked
   Name: {{contact.name}} — Reservation — {{contact.last_call_date}}

2. Send SMS to Admin
   ─────────────────────────────────────────────────
   📅 NEW RESERVATION — The Greek Grill
   Name: {{contact.name}}
   Phone: {{contact.phone}}
   Date/Time: {{contact.last_ready_time}}
   Called: {{contact.last_call_date}} at {{contact.last_call_time}}
   ─────────────────────────────────────────────────

3. Send SMS to Customer
   ─────────────────────────────────────────────────
   Hi {{contact.name}}! Your table at The Greek Grill is booked
   for {{contact.last_ready_time}}. We look forward to seeing you!
   📍 #31 99 Wye Rd, Sherwood Park, AB
   ─────────────────────────────────────────────────

4. [Day-before reminder — future]
   Wait until 1 day before reservation date
   Send SMS to Customer: "Reminder: your table at The Greek Grill
   is tomorrow at [time]. See you then!"
```

---

### WF-03 — Info Only
**Trigger:** Tag added: `info-only`
**Purpose:** Caller asked questions but didn't order — warm follow-up

```
1. Wait: 3 hours

2. Send SMS to Customer
   ─────────────────────────────────────────────────
   Hi {{contact.name}}! Thanks for calling The Greek Grill.
   We're open Mon–Thu 9 AM–10 PM and Fri–Sun 9 AM–midnight.
   Ready to order? Give us a call — we'd love to have you! 🍽️
   ─────────────────────────────────────────────────

[No opportunity created — no action taken on this call]
```

---

### WF-04 — Transferred
**Trigger:** Tag added: `transferred`
**Purpose:** Caller was handed to a human — alert staff immediately

```
1. Send SMS to Admin (immediate, no wait)
   ─────────────────────────────────────────────────
   ⚠️ CALL TRANSFERRED — The Greek Grill
   A caller was transferred to staff.
   Name: {{contact.name}}
   Phone: {{contact.phone}}
   Called at: {{contact.last_call_time}}
   ─────────────────────────────────────────────────

[No opportunity created until staff handles the call]
```

---

### WF-07 — Missed Call
**Trigger:** Missed Call (GHL native — completely separate from Call Ended)
**Runs:** When the voice agent never picked up, or call was rejected
**Does NOT overlap with WF-00** — this fires before the call ends, or when no call session started at all

```
STEP 1 — Wait 2 minutes
(Give the caller a chance to call back themselves)

STEP 2 — Check: Did they call back in those 2 minutes?
   ├── YES (new call started) → EXIT, WF-00 will handle it
   └── NO  → continue

STEP 3 — Apply tag: missed-call

STEP 4 — Send SMS to Customer
   ─────────────────────────────────────────────────
   Hi! We missed your call at The Greek Grill 😊
   We'd love to help — give us a ring whenever
   you're ready!
   📞 [restaurant phone number]
   Open Mon–Thu 9 AM–10 PM | Fri–Sun 9 AM–midnight
   ─────────────────────────────────────────────────

STEP 5 — If new contact (no prior record):
   Send SMS to Admin
   ─────────────────────────────────────────────────
   📵 MISSED CALL — New Contact
   Phone: {{contact.phone}}
   Time: {{now}}
   No prior record — may need manual follow-up.
   ─────────────────────────────────────────────────
```

> **Why the 2-minute wait?** Many callers redial immediately. Without the wait, they'd receive a "we missed you" SMS while already on the phone with Sofia — which looks broken. The 2-minute buffer handles this cleanly.
> **No opportunity created** — a missed call is not a confirmed intent. Only tag it and send the SMS.

---

### WF-05 — Re-engagement (Win-Back)
**Trigger:** Enrolled directly from WF-01 (step 8) at the end of every confirmed order
**Purpose:** Bring back customers who haven't ordered in 10+ days
**Goal (exit condition):** Contact gets `order-confirmed` tag again → exit immediately, cancel all pending steps

```
[Enrolled from WF-01 when order is confirmed]
    │
    ▼
Goal set: IF tag order-confirmed is added → EXIT this workflow (customer re-ordered)
    │
    ▼
Wait: 10 days
    │
    ▼
Still no new order?
    │
    ├── Apply tag: inactive-10-days
    │
    └── Send SMS to Customer — Wave 1
        ─────────────────────────────────────────────────
        Hey {{contact.name}}! It's been a while — we miss you
        at The Greek Grill 🍽️ Come back and try something new!
        We're open Mon–Thu 9 AM–10 PM, Fri–Sun 9 AM–midnight.
        Give us a call to order: [phone number]
        ─────────────────────────────────────────────────
    │
    ▼
Wait: 7 more days (total: 17 days since last order)
    │
    ▼
Still no new order?
    │
    ├── Apply tag: inactive-17-days
    │
    └── Send SMS to Customer — Wave 2 (stronger nudge)
        ─────────────────────────────────────────────────
        Hi {{contact.name}}! We haven't seen you in a bit —
        hope everything's good! 😊 Our kitchen is ready whenever
        you are. Call us at [phone number] to place an order.
        The Greek Grill — #31 99 Wye Rd, Sherwood Park.
        ─────────────────────────────────────────────────
    │
    ▼
Exit workflow (two waves sent — done)
```

> **Why 10 days?** A customer who orders weekly will re-trigger the goal and exit cleanly. 10 days catches true inactivity without over-messaging regulars.
> **Scalability note:** If you add a loyalty offer (e.g., "free drink on your next order"), add it to Wave 2 only — never Wave 1. Wave 1 is a soft nudge; Wave 2 is the closer.

---

### WF-06 — Google Review Request
**Trigger:** Tag added: `review-pending` (applied by WF-01 step 9)
**Purpose:** Capture positive reviews on Google; catch negative feedback privately before it goes public

```
[Tag: review-pending applied after order confirmed]
    │
    ▼
Wait: 3 hours
(enough time for the customer to have received and eaten their order)
    │
    ▼
Apply tag: review-requested
    │
    ▼
Send SMS to Customer — Review Request
    ─────────────────────────────────────────────────
    Hi {{contact.name}}! Hope you enjoyed your order from
    The Greek Grill 🍽️ How was your experience today?

    Reply HAPPY if we nailed it 👍
    Reply HELP if something wasn't right 👎

    Your feedback means everything to us!
    ─────────────────────────────────────────────────
    │
    ▼
Wait for reply (GHL keyword trigger — up to 24 hours)
    │
    ├── Reply: HAPPY  ──────────────────────────────────────────────→ WF-06a
    │
    └── Reply: HELP   ──────────────────────────────────────────────→ WF-06b
```

**WF-06a — Positive Response**
**Trigger:** Keyword: `HAPPY` (inbound SMS)

```
1. Apply tag: review-positive

2. Send SMS to Customer
   ─────────────────────────────────────────────────
   That's amazing to hear, {{contact.name}}! 🙌
   If you have a moment, we'd love a Google review —
   it helps us so much:
   [Your Google Review Link]
   Thank you for choosing The Greek Grill! ❤️
   ─────────────────────────────────────────────────
```

**WF-06b — Negative Response**
**Trigger:** Keyword: `HELP` (inbound SMS)

```
1. Apply tag: review-negative

2. Send SMS to Customer (immediate — do not delay)
   ─────────────────────────────────────────────────
   We're so sorry to hear that, {{contact.name}} 😔
   We want to make it right. A member of our team will
   reach out to you shortly. Thank you for letting us know.
   — The Greek Grill
   ─────────────────────────────────────────────────

3. Send SMS to Admin (immediate)
   ─────────────────────────────────────────────────
   ⚠️ NEGATIVE REVIEW ALERT — The Greek Grill
   Customer: {{contact.name}}
   Phone: {{contact.phone}}
   Receipt: {{contact.order_details}}
   Date: {{contact.last_call_date}}
   Action needed: Call or SMS this customer ASAP.
   ─────────────────────────────────────────────────
```

> **Why keywords instead of a rating scale?** HAPPY/HELP is simpler and faster to reply by SMS — a 1–5 rating requires typing a number and is easier to ignore. Keyword replies also trigger GHL automations natively without extra configuration.
> **No reply within 24 hours?** Exit silently. Do not send a second review request — one ask is enough. The `review-requested` tag prevents re-sending.

---

### WF-08 — Receipt Webhook (Inbound from n8n)
**Trigger:** Inbound Webhook (GHL native — fired by n8n Node 6)
**Purpose:** Receives receipt URL from n8n after PDF is generated → finds the contact by phone → writes URL to `order_details` field

```
STEP 1 — Inbound Webhook fires
   Payload received from n8n Node 6:
   {
     "receipt_url":    "https://...",
     "order_id":       "1777470908216",
     "customer_phone": "+919413752688"
   }

STEP 2 — Find Contact
   Lookup by: customer_phone (from webhook payload)

STEP 3 — Update Contact Field
   Field: order_details
   Value: {{receipt_url}}   ← mapped from webhook payload

STEP 4 — (Optional future) Update Opportunity
   Find open opportunity for this contact
   Add receipt URL to notes or custom field
```

> **`{{contact.order_details}}`** is the clickable PDF receipt URL used in all customer and admin SMS templates. Anyone who receives it can open the full receipt (items, GST, total) directly.
> This workflow runs in parallel with the call ending — it is NOT part of WF-00. It fires as soon as n8n Node 6 POSTs, which is right after the PDF is generated.

---

## Scalability Rules

These rules keep the system clean as you add new workflows:

1. **One trigger per workflow** — each sub-workflow is triggered by one tag only. Never trigger a workflow from two different events — that leads to double-firing.

2. **WF-00 is routing only** — never add business logic (SMSes, opportunities) to the master router. If you add a new call outcome, you add a new tag + a new workflow, and update WF-00 to apply the new tag. Everything else stays untouched.

3. **Tags accumulate, they don't reset** — a contact tagged `order-confirmed` 10 times just has the tag once. Use GHL's "workflow has run X times" or order count logic for milestone tags (e.g., VIP).

4. **All CRM data comes from contact fields** — never hardcode order details inside a workflow. Always pull from `{{contact.last_order_items}}`, `{{contact.last_ready_time}}` etc. This means when fields update, notifications update automatically.

5. **Admin notifications are always immediate** — never add a wait before admin SMS/email. Customer follow-ups can wait; the kitchen cannot.

6. **Always guard before routing** — WF-00 must check `last_order_status` before doing anything. Never apply tags or create opportunities based on a call ending alone — check that Sofia actually completed an action first.

7. **Missed call is a separate trigger, not a branch** — never try to detect missed calls inside WF-00. GHL has a native missed call trigger; use it. Mixing call-ended logic with missed-call logic in one workflow causes race conditions and double-firing.

---

## What to Build First (Suggested Order)

| Priority | Workflow | What to build | Why |
|----------|----------|--------------|-----|
| 1 | WF-00 | Master Router **with guard clause** | Everything depends on this — guard is non-negotiable |
| 2 | WF-07 | Missed Call | Runs on separate trigger — quick to build, immediate value |
| 3 | WF-01 | Admin SMS (order confirmed) | Kitchen needs to know immediately |
| 4 | WF-01 | Customer confirmation SMS | Order receipt for customer |
| 5 | WF-01 | Opportunity creation (Restaurant Orders pipeline) | CRM visibility |
| 6 | WF-04 | Transferred — admin SMS | Operational alert |
| 7 | WF-06 | Google Review request + WF-06a/06b responses | Revenue impact — reviews drive new customers |
| 8 | WF-02 | Reservation Booked | Table management |
| 9 | WF-05 | Re-engagement win-back (Wave 1 + Wave 2) | Retention — build once you have volume |
| 10 | WF-03 | Info Only follow-up | Low priority nurture |
| 11 | WF-01 | VIP milestone tag | Loyalty — add after enough orders are tracked |

---

*End of Post-Call Workflows Roadmap — The Greek Grill AB*
