# Tags — The Greek Grill AB
> GHL Voice Agent | Full Tag Taxonomy

---

## Status Tags
> Applied per call by WF-00 Master Router

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

---

## Order Sub-Tags
> Applied inside WF-01 after an order is confirmed

| Tag | Applied When | Purpose |
|-----|-------------|---------|
| `pickup-order` | `last_order_type` = pickup | Filter/report pickup volume |
| `delivery-order` | `last_order_type` = delivery | Filter/report delivery volume |
| `has-special-request` | `special_requests` is not empty | Kitchen awareness, allergy tracking |
| `review-pending` | End of every WF-01 | Triggers WF-06 (review request after 3 hrs) |

---

## Review Tags
> Applied inside WF-06

| Tag | Applied When | Purpose |
|-----|-------------|---------|
| `review-requested` | Review SMS sent to customer | Track who has been asked |
| `review-positive` | Customer replied HAPPY or 👍 | Potential Google review candidate |
| `review-negative` | Customer replied HELP or 👎 | Alert admin, handle complaint |

---

## Re-engagement Tags
> Applied inside WF-05

| Tag | Applied When | Purpose |
|-----|-------------|---------|
| `inactive-10-days` | No new order for 10 days | Wave 1 win-back SMS sent |
| `inactive-17-days` | No new order for 17 days | Wave 2 win-back SMS sent |

---

## Lifecycle Tags
> Applied by milestone logic

| Tag | Applied When | Purpose |
|-----|-------------|---------|
| `vip-customer` | Contact has 5+ confirmed orders | Priority treatment, future loyalty perks |
| `delivery-customer` | Has placed 2+ delivery orders | Delivery-specific promotions |

---

## Rules

- Tags accumulate — they do not reset. A contact tagged `order-confirmed` 10 times just has the tag once.
- Each workflow is triggered by exactly one tag. Never trigger a workflow from two different tags.
- The pre-call reset workflow (WF-pre) clears per-call tags at the start of every new call to prevent conflicts.

---

*End of Tags — The Greek Grill AB*
