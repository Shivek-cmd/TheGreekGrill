# Tags — The Greek Grill AB
> GHL Voice Agent | Full Tag Taxonomy

---

## Tag Lifecycle Rules

- **Per-call tags** — removed by WF-pre at the start of every new call
- **Lifetime tags** — never removed; they accumulate and mark history
- GHL only fires a workflow when a tag is **added** (not when it already exists) — so per-call tags must be removed after each call so they can re-trigger cleanly next time

---

## Status Tags — Per-Call
> Applied by WF-00 after the call ends. Removed by WF-pre at the start of the next call.

| Tag | Added By | Removed By | Purpose |
|-----|----------|------------|---------|
| `incomplete-call` | WF-00 Step 1 (guard) | WF-pre | Call ended with no action — blocks routing |
| `missed-call` | WF-07 Step 3 | WF-pre | Triggers callback SMS |
| `returning-customer` | WF-01 Step 6 | WF-pre | Per-call status — customer came back |
| `order-confirmed` | WF-00 Step 2 | WF-pre | Triggers WF-01 |
| `reservation-booked` | WF-00 Step 2 | WF-pre | Triggers WF-02 |
| `info-only` | WF-00 Step 2 | WF-pre | Triggers WF-03 |
| `transferred` | WF-00 Step 2 | WF-pre | Triggers WF-04 |

---

## Order Sub-Tags — Per-Call
> Applied inside WF-01. Removed by WF-pre at the start of the next call.

| Tag | Added By | Removed By | Purpose |
|-----|----------|------------|---------|
| `pickup-order` | WF-01 Step 1 | WF-pre | Filter/report pickup volume |
| `delivery-order` | WF-01 Step 1 | WF-pre | Filter/report delivery volume |
| `has-special-request` | WF-01 Step 2 | WF-pre | Kitchen awareness, allergy tracking |
| `review-pending` | WF-01 Step 11 | WF-06 (after SMS sent) | Triggers WF-06 — removed after use so it re-triggers on next order |

---

## Review Tags — Lifetime
> Applied inside WF-06. Never removed.

| Tag | Added By | Removed By | Purpose |
|-----|----------|------------|---------|
| `review-requested` | WF-06 (after 3hr wait) | Never | Permanent guard — customer already received review request, skip future ones |
| `review-positive` | WF-06a | Never | Historical — customer responded positively |
| `review-negative` | WF-06b | Never | Historical — customer complained |

---

## Re-engagement Tags — Semi-Lifetime
> Applied by WF-05 during inactivity. Removed by WF-01 when customer re-orders.

| Tag | Added By | Removed By | Purpose |
|-----|----------|------------|---------|
| `inactive-10-days` | WF-05 (Wave 1 sent) | WF-01 Step 0 | Customer was inactive 10 days — cleaned up on re-order |
| `inactive-17-days` | WF-05 (Wave 2 sent) | WF-01 Step 0 | Customer was inactive 17 days — cleaned up on re-order |

---

## Lifetime Tags — Never Removed
> Applied by milestone logic. Accumulate permanently.

| Tag | Added By | Removed By | Purpose |
|-----|----------|------------|---------|
| `new-customer` | WF-01 Step 6 (order_count = 1) | WF-01 Step 6 (order_count > 1) | Applied on first order, removed on second order — prevents new-customer campaigns from targeting returning customers |
| `vip-customer` | WF-01 Step 6 (order_count ≥ 5) | Never | 5+ confirmed orders — priority treatment |
| `delivery-customer` | WF-01 Step 6 (delivery_count = 2) | Never | 2+ delivery orders — delivery-specific promotions |

---

*End of Tags — The Greek Grill AB*
