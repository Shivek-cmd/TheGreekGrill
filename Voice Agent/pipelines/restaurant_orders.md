# Pipeline — Restaurant Orders
> The Greek Grill AB | GHL CRM Pipeline

---

## Overview

Created automatically by WF-01 when an order is confirmed. Tracks every order from placement to completion.

**Opportunity name format:** `[Name] — [Type] — [Date]`
**Example:** `Jessica — Delivery — 2026-04-28`

---

## Stages

| Stage | Who Moves It | When |
|-------|-------------|------|
| **New Order** | Auto (WF-01) | Opportunity created at call end |
| **In Preparation** | Manual (staff) | Kitchen starts working on it |
| **Ready** | Manual (staff) | Order is ready for pickup / dispatched |
| **Completed** | Manual or Auto | Order picked up / delivered |
| **Cancelled** | Manual (staff) | Customer cancelled or no-show |

---

## Notes

- Opportunity value: monetary value added manually or via future automation
- `order_details` field on the contact holds the clickable PDF receipt URL — `{{contact.order_details}}`

---

*End of Restaurant Orders Pipeline — The Greek Grill AB*
