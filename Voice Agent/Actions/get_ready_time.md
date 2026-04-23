# Get Ready Time — Custom Action
> The Greek Grill AB | GHL Voice Agent → n8n → Returns pickup time slots

---

## Overview

When Sofia reaches STEP 7 of the order flow, she triggers this custom action. n8n calculates the current Edmonton time server-side and returns 3 valid pickup slots. Sofia reads the result and speaks it once. No self-calculation — 100% accurate every time.

---

## Step 1 — GHL Custom Action Setup

**Action Type:** Custom Action 2.0
**Method:** `POST`
**Endpoint URL:**
```
https://n8n.bizbull.ai/webhook/get-ready-time-greek-grill
```
> Create a new webhook node in n8n. Use this path or any path you prefer — just match it here.

---

## Step 2 — GHL Custom Action Fields (Full UI Config)

Fill in every field in the GHL Custom Action form exactly as below.

---

**Action Name** *(0/64 char limit)*
```
get_ready_time
```

---

**When should the Custom API action take place?** *(0/500 char limit)*
```
When the agent has confirmed all items, spice levels, special requests, order type, and the caller's name and phone number — and is now ready to offer pickup time slots.
```

---

**What to say before performing the action** *(0/500 char limit)*
```
Let me check the times real quick...
```

---

**Run in background:** `OFF`
> Must be OFF — Sofia needs the time slots before she can continue speaking.

**Retry on failure:** `ON`
> If n8n times out, GHL will retry once automatically.

**API Timeout (sec):** `10`
> 10 seconds is enough. n8n responds in under 1 second for this action.

---

## Step 3 — Parameters (Sent as Query)

Only one parameter needed — for logging:

| # | Parameter Name | Type | In | Required | Description | Example |
|---|----------------|------|----|----------|-------------|---------|
| 1 | `restaurant_name` | String | Body | No | Always: `The Greek Grill` | `The Greek Grill` |

> n8n calculates everything server-side. No time or date needs to be sent from GHL.

---

## Step 4 — n8n Webhook Node Settings

**n8n workflow nodes:**
```
Node 1 — Webhook        (receives the POST from GHL)
Node 2 — Code           (calculates Edmonton time → returns slots)
Node 3 — Respond to Webhook  (sends JSON back to GHL)
```

> ⚠️ You MUST add a **Respond to Webhook** node as the last node. Without it, GHL will wait until the n8n timeout and Sofia will get no result.

**Webhook Node settings:**
| Setting | Value |
|---------|-------|
| HTTP Method | `POST` |
| Path | `get-ready-time-greek-grill` |
| Response Mode | `Using Respond to Webhook Node` |
| Authentication | None |

**Respond to Webhook Node settings:**
| Setting | Value |
|---------|-------|
| Respond With | `JSON` |
| Response Body | `{{ $json }}` *(passes the Code node output directly)* |
| Response Code | `200` |

---

## Step 5 — n8n Code Node (JavaScript)

```javascript
// Restaurant hours: 12:00 PM (open) to 2:00 AM (close)
// Slots are 30 minutes apart, first slot at least 30 min from now, rounded to :00 or :30

const now = new Date();

// Get current Edmonton time in minutes since midnight
const parts = new Intl.DateTimeFormat('en-CA', {
  timeZone: 'America/Edmonton',
  hour:   'numeric',
  minute: '2-digit',
  hour12: false
}).formatToParts(now);

const currentHour   = parseInt(parts.find(p => p.type === 'hour').value);
const currentMinute = parseInt(parts.find(p => p.type === 'minute').value);
const currentTotal  = currentHour * 60 + currentMinute;

// Restaurant: open 720 min (12:00 PM), close 120 min (2:00 AM next day)
// Minutes 120–719 = closed (2:00 AM to 11:59 AM)
// Minutes 720–1439 = open evening/night
// Minutes 0–119   = open late night (past midnight to 1:59 AM)

function roundUpToSlot(minutes) {
  const remainder = minutes % 30;
  return remainder === 0 ? minutes : minutes + (30 - remainder);
}

function minutesToTime(minutes) {
  const m = ((minutes % 1440) + 1440) % 1440; // normalise
  const h = Math.floor(m / 60);
  const min = m % 60;
  const period = h >= 12 ? 'PM' : 'AM';
  const display = h === 0 ? 12 : h > 12 ? h - 12 : h;
  return `${display}:${min.toString().padStart(2, '0')} ${period}`;
}

function isOpen(minutes) {
  const m = ((minutes % 1440) + 1440) % 1440;
  // Valid: 12:00 PM–11:59 PM (720–1439) or 12:00 AM–1:59 AM (0–119)
  // Closing at exactly 2:00 AM (120) — not a valid slot
  return (m >= 720 && m <= 1439) || (m >= 0 && m < 120);
}

const DEFAULT_SLOTS = [750, 780, 810]; // 12:30 PM, 1:00 PM, 1:30 PM (before/after hours fallback)

let rawSlots = [];
let status   = '';

const isClosed = currentTotal >= 120 && currentTotal < 720;

if (isClosed) {
  // Before opening or after closing — offer first 3 slots after opening
  rawSlots = DEFAULT_SLOTS;
  status   = currentTotal < 720 ? 'before_opening' : 'after_closing';
} else {
  // Restaurant is open — calculate from now
  const earliest = roundUpToSlot(currentTotal + 30);

  for (let i = 0; i < 3; i++) {
    const candidate = earliest + i * 30;
    if (isOpen(candidate)) rawSlots.push(candidate);
  }

  if (rawSlots.length === 0) {
    // No slots fit before 2:00 AM — offer tomorrow's opening slots
    rawSlots = DEFAULT_SLOTS;
    status   = 'near_closing';
  } else {
    status = 'open';
  }
}

const slots = rawSlots.map(minutesToTime);

return [{
  json: {
    status,
    slot_1: slots[0] || null,
    slot_2: slots[1] || null,
    slot_3: slots[2] || null
  }
}];
```

---

## Step 6 — n8n Output

```json
{
  "status": "open",
  "slot_1": "2:00 PM",
  "slot_2": "2:30 PM",
  "slot_3": "3:00 PM"
}
```

**Status values:**

| Status | Meaning | Slots returned |
|--------|---------|----------------|
| `open` | Restaurant is open, slots calculated from now | 3 slots from current time |
| `before_opening` | Call before 12:00 PM | 12:30 PM, 1:00 PM, 1:30 PM |
| `after_closing` | Call after 2:00 AM | 12:30 PM, 1:00 PM, 1:30 PM |
| `near_closing` | Open but no slots fit before 2:00 AM | 12:30 PM, 1:00 PM, 1:30 PM (next opening) |

---

## Step 7 — How Sofia Uses the Response

In GHL, map the action response variables:
- `slot_1` → variable Sofia references as the first time option
- `slot_2` → second option
- `slot_3` → third option

Sofia says (STEP 7 of order flow):
> "We can have it ready at [slot_1], [slot_2], or [slot_3] — which works better for you?"

If only 1 or 2 slots returned (near closing): speak only those.

---

## Example Scenarios

| Edmonton Time | Status | Slots Offered |
|---------------|--------|---------------|
| 11:35 AM | `before_opening` | 12:30 PM, 1:00 PM, 1:30 PM |
| 1:45 PM | `open` | 2:30 PM, 3:00 PM, 3:30 PM |
| 1:15 AM | `open` | 1:45 AM → rounds to 2:00 AM — near closing → 12:30 PM, 1:00 PM, 1:30 PM |
| 2:30 AM | `after_closing` | 12:30 PM, 1:00 PM, 1:30 PM |

---

*End of Get Ready Time Action — The Greek Grill AB*
