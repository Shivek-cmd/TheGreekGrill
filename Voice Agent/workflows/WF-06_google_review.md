# WF-06 — Google Review Request
> The Greek Grill AB | GHL Workflow

---

**Trigger:** Tag added: `review-pending` (applied by WF-01 Step 9)
**Purpose:** Capture positive reviews on Google; catch negative feedback privately before it goes public

---

## Main Flow

```
[Tag: review-pending applied after order confirmed]
    │
    ▼
Wait: 3 hours
(enough time for customer to receive and eat their order)
    │
    ▼
Apply tag: review-requested
    │
    ▼
Send SMS to Customer
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
    ├── Reply: HAPPY ──→ WF-06a
    └── Reply: HELP  ──→ WF-06b
```

> No reply within 24 hours → exit silently. Do not send a second request. The `review-requested` tag prevents re-sending.

---

## WF-06a — Positive Response

**Trigger:** Keyword: `HAPPY` (inbound SMS)

```
STEP 1 — Apply tag: review-positive

STEP 2 — Send SMS to Customer
   ─────────────────────────────────────────────────
   That's amazing to hear, {{contact.name}}! 🙌
   If you have a moment, we'd love a Google review —
   it helps us so much:
   [Your Google Review Link]
   Thank you for choosing The Greek Grill! ❤️
   ─────────────────────────────────────────────────
```

---

## WF-06b — Negative Response

**Trigger:** Keyword: `HELP` (inbound SMS)

```
STEP 1 — Apply tag: review-negative

STEP 2 — Send SMS to Customer (immediate)
   ─────────────────────────────────────────────────
   We're so sorry to hear that, {{contact.name}} 😔
   We want to make it right. A member of our team will
   reach out to you shortly. Thank you for letting us know.
   — The Greek Grill
   ─────────────────────────────────────────────────

STEP 3 — Send SMS to Admin (immediate)
   ─────────────────────────────────────────────────
   ⚠️ NEGATIVE REVIEW ALERT — The Greek Grill
   Customer: {{contact.name}}
   Phone:    {{contact.phone}}
   Receipt:  {{contact.order_details}}
   Date:     {{contact.last_call_date}}
   Action needed: Call or SMS this customer ASAP.
   ─────────────────────────────────────────────────
```

---

## Notes

- **Why HAPPY/HELP instead of 1–5 rating?** Keyword replies are faster to send by SMS and trigger GHL automations natively — no extra configuration needed.
- **`review-requested` tag** prevents the same customer from being asked twice, even across multiple orders.

---

*End of WF-06 — The Greek Grill AB*
