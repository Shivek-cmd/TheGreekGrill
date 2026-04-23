# Order Receipt — Custom Action 2.0
> The Greek Grill AB | GHL Voice Agent → n8n Webhook Flow

---

## Overview

When Sofia finalizes an order, GHL triggers this Custom Action 2.0. It fires a POST request to n8n with 10 query parameters. n8n parses the data, calculates the total, and returns a structured order object ready for receipt generation or CRM update.

---

## Step 1 — GHL Custom Action Setup

**Action Type:** Custom Action 2.0
**Method:** `POST`
**Endpoint URL:**
```
https://n8n.bizbull.ai/webhook/4dcea672-3074-4914-80fb-57e7cbbec2fe
```

---

## Step 2 — Parameters (Sent as Query)

> All 10 parameters are sent as query params on the POST request.

| # | Parameter Name | Type | Description | Example |
|---|----------------|------|-------------|---------|
| 1 | `items_summary` | String | All ordered items packed as `quantity:name:price:notes` per item, separated by `\|` | `1:Butter Chicken:19.99:spicy\|3:Plain Naan:3.49:none` |
| 2 | `order_type` | String | Pickup (takeout), delivery, or dine-in (table reservation) | `pickup` |
| 3 | `item_1_notes` | String | Spice level or special note for first item | `spicy` |
| 4 | `delivery_address` | String | Full delivery address if delivery — leave empty otherwise | `123 Main St, Edmonton, AB` |
| 5 | `buzzer_code` | String | Buzzer or entry code if delivery — leave empty otherwise | `#302` |
| 6 | `requested_time` | String | Ready time the caller agreed to | `2:15 PM` |
| 7 | `customer_name` | String | Name given for the order | `Sam` |
| 8 | `customer_phone` | String | Caller's confirmed phone number | `+16476150677` |
| 9 | `restaurant_name` | String | Always: `The Greek Grill` | `The Greek Grill` |
| 10 | `restaurant_address` | String | Always: `#31 99 Wye Rd, Sherwood Park, AB T8B 1M1` | `#31 99 Wye Rd, Sherwood Park, AB T8B 1M1` |

---

### items_summary Format

```
quantity:name:price:notes
```

Multiple items separated by `\|` (backslash + pipe):

```
1:Rack of Lamb:38.95:medium\|2:Greek Salad:16.95:none\|1:Chocolate Lava Cake:8.95:none
```

> This matches the format referenced in the agent prompt GUIDELINES.

---

## Step 3 — n8n Workflow

**Node 1:** Webhook (receives the POST request)
**Node 2:** Code node (JavaScript — parses and structures the data)

### JavaScript Code (Node 2)

```javascript
const raw = $input.first();

// GHL sends form-encoded data — fields live under raw.json.query
const input = raw.json.query;

// items_summary format: "qty:name:price:notes" per item
// GHL sends the pipe separator as \| (literal backslash+pipe) — handle both
const itemsRaw = input.items_summary || '';
const items = itemsRaw
  .split(/\\\||\|/)          // split on \| or | — handles both cases
  .map(s => s.trim())
  .filter(s => s !== '')
  .map(s => {
    const parts = s.split(':');
    const quantity = parts[0];
    const name     = parts[1];
    const price    = parts[2];
    const notes    = parts.slice(3).join(':'); // rejoin safely if notes had a colon
    return {
      name:     (name     || '').trim(),
      quantity: parseInt(quantity)   || 1,
      price:    parseFloat(price)    || 0,
      notes:    (notes    || '').trim()
    };
  });

const totalPrice = items.reduce((sum, i) => sum + (i.price * i.quantity), 0);

return [{
  json: {
    order: {
      items,
      orderType:       input.order_type       || null,
      deliveryAddress: input.delivery_address || null,
      buzzerCode:      input.buzzer_code      || null,
      allergies:       input.allergies        || null,
      requestedTime:   input.requested_time   || null,
      totalPrice:      parseFloat(totalPrice.toFixed(2))
    },
    customer: {
      name:  input.customer_name  || null,
      phone: input.customer_phone || null
    },
    restaurant: {
      name:       input.restaurant_name    || null,
      locationId: input.restaurant_name    || null,
      address:    input.restaurant_address || null
    }
  }
}];
```

---

## Step 4 — n8n Output (Structured JSON)

The Code node returns this structure to the next node in the workflow:

```json
{
  "order": {
    "items": [
      { "name": "Cheese Omelette", "quantity": 1, "price": 13.99, "notes": "none" },
      { "name": "Cappuccino",      "quantity": 1, "price": 4.95,  "notes": "none" },
      { "name": "Plain Pancakes",  "quantity": 1, "price": 11.99, "notes": "none" }
    ],
    "orderType":       "pickup",
    "deliveryAddress": null,
    "buzzerCode":      null,
    "allergies":       null,
    "requestedTime":   "1:00 PM",
    "totalPrice":      30.93
  },
  "customer": {
    "name":  "Jessica",
    "phone": "+91 94137 52688"
  },
  "restaurant": {
    "name":       "The Greek Grill",
    "locationId": "The Greek Grill",
    "address":    "#31 99 Wye Rd, Sherwood Park, AB T8B 1M1"
  }
}
```

---

## Notes

- `totalPrice` is auto-calculated by n8n from `quantity × price` per item — do not send it from GHL.
- `restaurant_name` and `restaurant_address` are hardcoded constants — Sofia always sends fixed values.
- If `delivery_address` is empty, n8n sets `deliveryAddress: null` — safe for pickup/dine-in orders.
- The `\|` pipe separator is a GHL quirk — the Code node handles both `\|` and `|` formats.

---

*End of Order Receipt Action — The Greek Grill AB*
