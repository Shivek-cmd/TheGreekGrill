# Pipeline — Table Reservations
> The Greek Grill AB | GHL CRM Pipeline

---

## Overview

Created automatically by WF-02 when a reservation is booked. Tracks every table booking from confirmation to completion.

**Opportunity name format:** `[Name] — Reservation — [Date]`
**Example:** `John — Reservation — 2026-05-01`

---

## Stages

| Stage | Who Moves It | When |
|-------|-------------|------|
| **Booked** | Auto (WF-02) | Opportunity created at call end |
| **Reminder Sent** | Auto (WF-02, day before) | Automated reminder triggered |
| **Arrived** | Manual (host) | Party seated |
| **Completed** | Manual (host) | Party finished and left |
| **No Show / Cancelled** | Manual (host) | Didn't arrive or cancelled |

---

*End of Table Reservations Pipeline — The Greek Grill AB*
