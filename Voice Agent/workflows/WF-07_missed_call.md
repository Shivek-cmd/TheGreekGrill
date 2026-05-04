# WF-07 — Missed Call
> The Greek Grill AB | GHL Workflow

---

**Trigger:** Missed Call (GHL native — completely separate from Call Ended)
**Runs:** When the voice agent never picked up, or the call was rejected
**Does NOT overlap with WF-00** — fires before the call ends, or when no call session started at all

---

## Steps

```
STEP 1 — Wait 2 minutes
(Give the caller a chance to call back themselves)

STEP 2 — Check: Did they call back in those 2 minutes?
   ├── YES (new call started) → EXIT — WF-00 will handle it
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
   Time:  {{now}}
   No prior record — may need manual follow-up.
   ─────────────────────────────────────────────────
```

---

## Notes

- **Why the 2-minute wait?** Many callers redial immediately. Without the wait, they'd receive a "we missed you" SMS while already on the phone with Sofia — which looks broken.
- **No opportunity created** — a missed call is not confirmed intent. Tag and SMS only.

---

*End of WF-07 — The Greek Grill AB*
