# Contact Field Updates — GHL Action
> The Greek Grill AB | Voice Agent → CRM Field Mapping

---

## Overview

GHL's **Update Contact Field** action runs at two points during/after the call:
- **During the call** — extracted in real time as Sofia collects information
- **After the call** — triggered once the call ends

Each field has one of two behaviours:
- **Save Value** — writes the value only if the field is currently empty (protects existing data)
- **Replace Value** — always overwrites with the latest value

---

## All Contact Fields

| # | Action Name | CRM Field | Timing | Behaviour | What Sofia Extracts | Example Values |
|---|-------------|-----------|--------|-----------|---------------------|----------------|
| 1 | Save First Name | `first_name` | During call | Replace | The name the caller provided for their order, transliterated into English | `John` `Sarah` |
| 2 | Save Contact Phone | `phone` | During call | Save | The phone number the caller verbally confirmed, read back digit by digit | `+15551234567` `+15559876543` |
| 3 | Save Order Type | `last_order_type` | During call | Replace | The type of order placed — pickup, delivery, or table reservation | `pickup` `delivery` `table reservation` |
| 4 | Save Delivery Address | `last_delivery_address` | During call | Replace | Full delivery address including street number, name, city, and unit if applicable | `123 Main St, Edmonton, AB` `45 Oak Ave Unit 3, Edmonton` |
| 5 | Save Order Items | `last_order_items` | During call | Replace | Plain text summary of all items with quantity and spice level where applicable | `2 Butter Chicken medium, 1 Garlic Naan` `1 Rack of Lamb spicy, 2 Greek Salad` |
| 6 | Save Ready Time | `last_ready_time` | During call | Replace | The confirmed pickup or delivery ready time the caller agreed to | `7:15 PM` `11:30 AM` `8:45 PM` |
| 7 | Save Special Requests | `special_requests` | During call | Save | Any allergies, dietary restrictions, or special preparation notes the caller mentioned | `No nuts` `Extra spicy` `Gluten-free only` |
| 8 | Save Buzzer Code | `buzzer_code` | During call | Replace | The building entry buzzer or door code provided for delivery | `#302` `4521` `B12` |
| 9 | Save Preferred Language | `preferred_language` | During call | Save | The language or mix of languages the caller used during the conversation | `English` `Hindi` `Hinglish` |
| 10 | Save Last Call Date | `last_call_date` | After call | Replace | Today's date — the date this call took place | `2026-04-23` `2026-05-01` |
| 11 | Save Last Call Time | `last_call_time` | After call | Replace | The local time this call took place (Edmonton timezone, 12-hour format) | `1:51 PM` `9:30 AM` |
| 12 | Save Order Status | `last_order_status` | After call | Replace | Whether the caller placed an order, booked a reservation, asked for info only, or was transferred | `Order Confirmed` `Reservation Booked` `Info Only` `Transferred` |
| 13 | — (GHL response mapping) | `last_receipt_url` | After call (n8n response) | Replace | Receipt PDF URL — returned by n8n Node 5, mapped via Finalize Order action response | `https://...` |
| 14 | — (WF-08 inbound webhook) | `order_details` | After call (n8n Node 6 → WF-08) | Replace | Clickable PDF receipt URL — written by WF-08 after n8n fires. Used in all SMS templates as `{{contact.order_details}}` | `https://...` |
| 15 | — (WF-01 Step 5) | `order_count` | After call (WF-01) | Increment | Lifetime order counter — incremented by 1 on every confirmed order. Never reset. Used to detect new-customer (=1) and VIP (≥5) | `1` `4` `7` |
| 16 | — (WF-01 Step 7) | `delivery_count` | After call (WF-01, delivery orders only) | Increment | Lifetime delivery counter — incremented only when order_type = delivery. Never reset. Used to apply delivery-customer tag at count = 2 | `1` `2` `5` |

---

## Timing Breakdown

### During the Call (9 fields — unchanged)
Sofia extracts and saves these in real time as the conversation progresses:

```
Step 1  → First Name extracted           → saves to first_name
Step 2  → Phone confirmed                → saves to phone
Step 3  → Order type decided             → saves to last_order_type
Step 4  → Delivery address collected     → saves to last_delivery_address (delivery only)
Step 5  → Buzzer code collected          → saves to buzzer_code (delivery only)
Step 6  → Items confirmed                → saves to last_order_items
Step 7  → Special requests collected     → saves to special_requests
Step 8  → Ready time agreed              → saves to last_ready_time
Step 9  → Language detected throughout   → saves to preferred_language
```

### After the Call (3 fields)
Triggered once the call ends:

```
After call → Date stamped                → saves to last_call_date
After call → Time stamped                → saves to last_call_time   ← feeds orderTimeLocal on receipt
After call → Call outcome determined     → saves to last_order_status
```

---

## Advanced Controls — "When and What to Update"

This is the GHL field instruction that tells the AI what to extract. Use these exact descriptions:

| Field | Instruction Text |
|-------|-----------------|
| `first_name` | The name the caller provided for their order, transliterated into English. |
| `phone` | The phone number the caller verbally confirmed during the call, read back digit by digit and verified as correct. |
| `last_order_type` | The type of order the caller placed — pickup (takeout), delivery, or table reservation (dine-in). |
| `last_delivery_address` | The full delivery address the caller provided, including street number, street name, city, and any unit number. |
| `last_order_items` | A plain text summary of all items the caller ordered, including quantity, item name, and spice level where applicable. |
| `last_ready_time` | The confirmed pickup or delivery ready time the caller agreed to during this call. |
| `special_requests` | Any allergies, dietary restrictions, or special preparation notes the caller mentioned during the order. |
| `buzzer_code` | The building entry buzzer or door code the caller provided for delivery access. |
| `preferred_language` | The language or mix of languages the caller used during the conversation. |
| `last_call_date` | Today's date — the date this call took place, in YYYY-MM-DD format. |
| `last_call_time` | The local time this call took place, in 12-hour format (America/Edmonton timezone). |
| `last_order_status` | Whether the caller successfully placed an order, booked a reservation, only asked for information, or was transferred to a human during this call. If the caller hung up, said nothing, or the call ended without any action being completed, leave this field empty. An empty value signals WF-00 to treat this as an incomplete call and skip all routing. |

---

## ⚠️ Gap — special_requests Missing from Webhook

`special_requests` is saved to the CRM correctly but is **not currently sent to the order receipt webhook**.

This means the PDF receipt always shows `Instructions: -` even when the caller mentioned an allergy or special request.

**Fix: add `special_requests` as parameter 13 in the order_receipt webhook.**

See [order_receipt.md](order_receipt.md) — the parameter and Node 2 update are documented there.

---

*End of Contact Field Updates — The Greek Grill AB*
