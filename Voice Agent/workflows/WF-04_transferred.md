# WF-04 — Transferred
> The Greek Grill AB | GHL Workflow

---

**Trigger:** Tag added: `transferred`
**Purpose:** Caller was handed to a human — alert staff immediately

---

## Steps

```
STEP 1 — Send SMS to Admin (immediate, no wait)
   ─────────────────────────────────────────────────
   ⚠️ CALL TRANSFERRED — The Greek Grill
   A caller was transferred to staff.
   Name:     {{contact.name}}
   Phone:    {{contact.phone}}
   Called at: {{contact.last_call_time}}
   ─────────────────────────────────────────────────
```

> No opportunity created — staff handles the call manually and decides next steps.

---

*End of WF-04 — The Greek Grill AB*
