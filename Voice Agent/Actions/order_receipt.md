# Order Receipt — Custom Action 2.0
> The Greek Grill AB | GHL Voice Agent → n8n → PDF → PrintNode

---

## Full n8n Flow

```
Node 1 — Webhook            (receives GHL POST request)
Node 2 — Code: GHL Parser   (parses query params → structured JSON)
Node 3 — Code: Data Mapper  (maps GHL structure → ordercontent format)  ← NEW
Node 4 — Code: Slip PDF     (generates HTML receipt → calls PDF API)
Node 5 — PrintNode          (sends PDF to printer)
```

---

## Step 1 — GHL Custom Action Setup

**Action Type:** Custom Action 2.0
**Method:** `POST`
**Endpoint URL:**
```
https://n8n.bizbull.ai/webhook/4dcea672-3074-4914-80fb-57e7cbbec2fe
```

**Action Name:**
```
finalize_order
```

**When should the Custom API action take place?** *(0/500 char limit)*
```
When the caller has confirmed the full order recap and said they are done — all items, order type, ready time, name, and phone are confirmed and the agent is ready to submit the order.
```

**What to say before performing the action** *(0/500 char limit)*
```
Give me just a second while I get that sorted for you...
```

**Run in background:** `OFF`
> Must be OFF — the receipt must be sent and printed before the call ends.

**Retry on failure:** `ON`

**API Timeout (sec):** `15`
> 15 seconds allows time for PDF generation and PrintNode submission.

---

## Step 2 — Parameters (Sent as Query)

> All 10 parameters are sent as query params on the POST request.

| # | Parameter Name | Type | Description | Example |
|---|----------------|------|-------------|---------|
| 1 | `items_summary` | String | All ordered items packed as `quantity:name:price:notes` per item, separated by `\|` | `1:Butter Chicken:19.99:spicy\|3:Plain Naan:3.49:none` |
| 2 | `order_type` | String | Pickup (takeout), delivery, or dine-in | `pickup` |
| 3 | `item_1_notes` | String | Spice level or special note for first item | `spicy` |
| 4 | `delivery_address` | String | Full delivery address if delivery — leave empty otherwise | `123 Main St, Edmonton, AB` |
| 5 | `buzzer_code` | String | Buzzer or entry code if delivery — leave empty otherwise | `#302` |
| 6 | `requested_time` | String | Ready time the caller agreed to | `2:15 PM` |
| 7 | `customer_name` | String | Name given for the order | `Sam` |
| 8 | `customer_phone` | String | Caller's confirmed phone number | `+16476150677` |
| 9 | `restaurant_name` | String | Always: `The Greek Grill` | `The Greek Grill` |
| 10 | `restaurant_address` | String | Always: `#31 99 Wye Rd, Sherwood Park, AB T8B 1M1` | `#31 99 Wye Rd, Sherwood Park, AB T8B 1M1` |
<!-- NOTE: order_date and order_time are NOT sent from GHL — both generated server-side in Node 3 from Edmonton timezone -->
| 13 | `special_requests` | String | Allergies, dietary restrictions, or special prep notes — leave empty if none | `No nuts` `Gluten-free only` |

---

### items_summary Format

```
quantity:name:price:notes
```

Multiple items separated by `\|`:

```
1:Rack of Lamb:38.95:medium\|2:Greek Salad:16.95:none\|1:Chocolate Lava Cake:8.95:none
```

---

## Node 2 — Code: GHL Parser (existing — no changes)

Receives the raw webhook, parses `items_summary`, calculates `totalPrice`.

```javascript
const raw = $input.first();
const input = raw.json.query;

function normalizePhone(raw) {
  if (!raw) return null;
  const wordMap = {
    'zero':'0','one':'1','two':'2','three':'3','four':'4',
    'five':'5','six':'6','seven':'7','eight':'8','nine':'9'
  };
  let n = raw.toLowerCase()
    .replace(/\b(zero|one|two|three|four|five|six|seven|eight|nine)\b/g, m => wordMap[m])
    .replace(/[^\d+]/g, '');
  if (n.startsWith('+')) return n;          // already has country code — leave as-is
  if (/^\d{10}$/.test(n)) return '+1' + n;  // 10 digits → North American
  if (/^\d{11,}$/.test(n)) return '+' + n;  // 11+ digits → has country code, just missing +
  return n || raw;
}

const itemsRaw = input.items_summary || '';
const items = itemsRaw
  .split(/\\\||\|/)
  .map(s => s.trim())
  .filter(s => s !== '')
  .map(s => {
    const parts = s.split(':');
    const quantity = parts[0];
    const name     = parts[1];
    const price    = parts[2];
    const notes    = parts.slice(3).join(':');
    const parsedPrice = parseFloat(price);
    return {
      name:     (name     || '').trim(),
      quantity: parseInt(quantity)   || 1,
      price:    isNaN(parsedPrice) || parsedPrice === 0 ? null : parsedPrice,
      notes:    (notes    || '').trim()
    };
  });

const totalPrice = items.reduce((sum, i) => sum + ((i.price || 0) * i.quantity), 0);

return [{
  json: {
    order: {
      items,
      orderType:       input.order_type       || null,
      deliveryAddress: input.delivery_address || null,
      buzzerCode:      input.buzzer_code      || null,
      allergies:       input.special_requests  || null,
      requestedTime:   input.requested_time   || null,
      // orderDateLocal and orderTimeLocal are NOT read from GHL — generated in Node 3
      totalPrice:      parseFloat(totalPrice.toFixed(2))
    },
    customer: {
      name:  input.customer_name              || null,
      phone: normalizePhone(input.customer_phone)
    },
    restaurant: {
      name:       input.restaurant_name    || null,
      locationId: input.restaurant_name    || null,
      address:    input.restaurant_address || null
    }
  }
}];
```

**Output from Node 2:**
```json
{
  "order": {
    "items": [...],
    "orderType": "pickup",
    "requestedTime": "1:00 PM",
    "totalPrice": 30.93
  },
  "customer": { "name": "Jessica", "phone": "+91 94137 52688" },
  "restaurant": { "name": "The Greek Grill", "address": "#31 99 Wye Rd, Sherwood Park, AB T8B 1M1" }
}
```

---

## Node 3 — Code: Data Mapper (NEW — add this between Node 2 and Slip PDF)

**Why this node exists:** The Slip PDF Processor was built for VAPI data (`ordercontent` wrapper, capital field names). GHL sends a different structure. This node maps GHL → the exact format Slip PDF expects — zero changes to Node 2 or Node 4.

```javascript
const data = $input.first().json;
const { order, customer, restaurant } = data;

// Generate Edmonton local date (GHL cannot extract this from the conversation)
const now = new Date();
const parts = new Intl.DateTimeFormat('en-CA', {
  timeZone: 'America/Edmonton',
  year: 'numeric', month: '2-digit', day: '2-digit',
  hour: '2-digit', minute: '2-digit', hour12: true
}).formatToParts(now);
const get = (type) => parts.find(p => p.type === type)?.value || '';
const orderDateLocal = `${get('year')}-${get('month')}-${get('day')}`;
const orderTimeLocal = `${get('hour')}:${get('minute')} ${get('dayPeriod')}`;

// Generate order ID from timestamp
const orderId = Date.now();

return [{
  json: {
    ordercontent: {
      restaurant_name:    restaurant.name,
      restaurant_address: restaurant.address,
      restaurant_email:   null,
      customer: {
        Name:     customer.name,
        Phone:    customer.phone,
        timezone: 'America/Edmonton',
        Email:    null
      },
      order: {
        Type:                order.orderType,
        items:               order.items,
        specialInstructions: order.allergies        || null,
        orderDateLocal,
        orderTimeLocal,
        fulfillmentTimeLocal: order.requestedTime
      },
      id: String(orderId)
    },
    order_id: orderId
  }
}];
```

**Output from Node 3 — matches exactly what Slip PDF Processor expects:**
```json
{
  "ordercontent": {
    "restaurant_name": "The Greek Grill",
    "restaurant_address": "#31 99 Wye Rd, Sherwood Park, AB T8B 1M1",
    "restaurant_email": null,
    "customer": {
      "Name": "Jessica",
      "Phone": "+91 94137 52688",
      "timezone": "America/Edmonton",
      "Email": null
    },
    "order": {
      "Type": "pickup",
      "items": [
        { "name": "Cheese Omelette", "quantity": 1, "price": 13.99, "notes": "none" },
        { "name": "Cappuccino",      "quantity": 1, "price": 4.95,  "notes": "none" },
        { "name": "Plain Pancakes",  "quantity": 1, "price": 11.99, "notes": "none" }
      ],
      "specialInstructions": null,
      "orderDateLocal": "2026-04-23",
      "orderTimeLocal": "01:00 PM",
      "fulfillmentTimeLocal": "1:00 PM"
    },
    "id": "1745123456789"
  },
  "order_id": 1745123456789
}
```

---

## Node 4 — Code: Slip PDF Processor (existing — no changes)

Uses the same code as the VAPI version. No modifications needed.

```javascript
const { customer, items, order, id, restaurant_address, restaurant_name } = $json.ordercontent;

const subtotal = order.items.reduce((sum, i) => {
  return sum + (Number(i.price || 0) * i.quantity);
}, 0);

const gst = subtotal * 0.05;
const total = subtotal + gst;

const html = `
<html>
<head>
<style>
  body { font-family: monospace; font-size: 40px; margin: 0; padding: 10px; color: #000; }
  .center { text-align: center; font-size: 44px; font-weight: bold; }
  .line { border-top: 2px dashed #000; margin: 16px 0; }
  .item { margin-bottom: 22px; padding-left: 6px; }
  .item-name { font-weight: bold; font-size: 40px; }
  .small { font-size: 36px; }
  .row { display: flex; justify-content: space-between; font-size: 36px; margin-top: 4px; }
  .right { text-align: right; font-size: 36px; }
  .bold { font-weight: bold; font-size: 36px; }
</style>
</head>
<body>

<div class="center">${restaurant_name}</div>
<div class="center small">${restaurant_address}</div>
<div class="center small">ORDER ID: ${$json?.order_id}</div>
<div class="center">ORDER RECEIPT</div>
<div class="line"></div>

<div class="small">
  <div><b>Name:</b> ${customer.Name}</div>
  <div><b>Phone:</b> ${customer.Phone}</div>
</div>

<div class="line"></div>
<div class="bold">ITEMS</div>
<div class="line"></div>

${order.items.map(i => {
  const price = Number(i.price || 0);
  const itemTotal = price * i.quantity;
  return `
  <div class="item">
    <div class="item-name">${i.name}</div>
    <div class="row">
      <span>Qty: ${i.quantity}</span>
      <span>Price: $${price.toFixed(2)}</span>
    </div>
    <div class="row bold">
      <span>Total</span>
      <span>$${itemTotal.toFixed(2)}</span>
    </div>
    ${i.notes && i.notes !== 'none' ? `<div class="small">Note: ${i.notes}</div>` : ''}
  </div>
  `;
}).join('')}

<div class="line"></div>

<div class="right">
  <div><b>Subtotal:</b> $${subtotal.toFixed(2)}</div>
  <div><b>GST (5%):</b> $${gst.toFixed(2)}</div>
  <div><b>TOTAL:</b> $${total.toFixed(2)}</div>
</div>

<div class="line"></div>

<div class="small">
  <div><b>Type:</b> ${order.Type}</div>
  <div><b>Instructions:</b> ${order.specialInstructions || 'None'}</div>
  <div><b>Date:</b> ${order.orderDateLocal}</div>
  <div><b>Time:</b> ${order.orderTimeLocal}</div>
  <div><b>Ready By:</b> ${order.fulfillmentTimeLocal}</div>
</div>

<div class="line"></div>
<div class="center">THANK YOU</div>

</body>
</html>
`;

const response = await this.helpers.httpRequest({
  method: 'POST',
  url: 'http://187.124.229.161:8000/generate.php',
  headers: { 'Content-Type': 'application/json' },
  body: { html },
  encoding: null
});

return [{ json: { id, url: response?.url || null } }];
```

---

## Node 5 — PrintNode (existing — no changes)

**URL:** `https://api.printnode.com/printjobs` (POST)
**Body:**
```json
{
  "printerId": 75369605,
  "title": "Order Receipt - {{$json.id}}",
  "contentType": "pdf_uri",
  "content": "{{$json.url}}",
  "source": "Printnode web 2.0"
}
```

---

## Notes

- **Only Node 3 is new.** Everything else is unchanged from the VAPI version.
- `order_id` is generated from `Date.now()` (Unix timestamp in ms) — unique per order.
- `orderDateLocal` and `orderTimeLocal` are sent from GHL as parameters 11 and 12 — not generated server-side.
- GST (5%) is calculated in Node 4 on the subtotal — not sent from GHL.
- `notes: "none"` from GHL is suppressed on the receipt — only real notes print.

---

*End of Order Receipt Action — The Greek Grill AB*
