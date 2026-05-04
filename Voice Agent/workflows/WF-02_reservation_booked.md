# WF-02 — Reservation Booked
> The Greek Grill AB | GHL Workflow

---

**Trigger:** Tag added: `reservation-booked`

---

## Steps

```
STEP 1 — Create Opportunity
   Pipeline: Table Reservations
   Stage:    Booked
   Name:     {{contact.name}} — Reservation — {{contact.last_call_date}}

STEP 2 — Send SMS to Admin (immediate)
   ─────────────────────────────────────────────────
   📅 NEW RESERVATION — The Greek Grill
   Name:    {{contact.name}}
   Phone:   {{contact.phone}}
   Date/Time: {{contact.last_ready_time}}
   Called:  {{contact.last_call_date}} at {{contact.last_call_time}}
   ─────────────────────────────────────────────────

STEP 3 — Send SMS to Customer
   ─────────────────────────────────────────────────
   Hi {{contact.name}}! Your table at The Greek Grill is booked
   for {{contact.last_ready_time}}. We look forward to seeing you!
   📍 #31 99 Wye Rd, Sherwood Park, AB
   ─────────────────────────────────────────────────

STEP 4 — Day-before reminder (future)
   Wait until 1 day before reservation date
   Send SMS to Customer:
   "Reminder: your table at The Greek Grill is tomorrow
   at [time]. See you then!"
```

---

*End of WF-02 — The Greek Grill AB*
