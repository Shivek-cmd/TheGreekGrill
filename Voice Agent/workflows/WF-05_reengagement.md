# WF-05 — Re-engagement (Win-Back)
> The Greek Grill AB | GHL Workflow

---

**Trigger:** Enrolled directly from WF-01 (Step 8) after every confirmed order
**Purpose:** Bring back customers who haven't ordered in 10+ days
**Goal (exit condition):** Contact gets `order-confirmed` tag again → exit immediately, cancel all pending steps

---

## Steps

```
[Enrolled from WF-01 when order is confirmed]
    │
    ▼
Goal set: IF tag order-confirmed is added → EXIT (customer re-ordered)
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

---

## Notes

- **Why 10 days?** A customer who orders weekly will re-trigger the goal and exit cleanly. 10 days catches true inactivity without over-messaging regulars.
- **Loyalty offer rule:** If adding a discount or free item, put it in Wave 2 only — never Wave 1. Wave 1 is a soft nudge; Wave 2 is the closer.

---

*End of WF-05 — The Greek Grill AB*
