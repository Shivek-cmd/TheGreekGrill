# Get Ready Time — Custom Action
> The Greek Grill AB | GHL Voice Agent → n8n → Returns minimum ready time

---

## Overview

When Sofia reaches STEP 7 of the order flow, she triggers this custom action. n8n calculates the current Edmonton time server-side and returns the **earliest possible ready time** (current time + 30 minutes, rounded to the next :00 or :30). Sofia tells the caller the minimum time and asks what time works for them — the caller's answer becomes `requested_time`. No self-calculation — 100% accurate every time.

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
When the agent has confirmed all items, spice levels, special requests, order type, and the caller's name and phone number — and is now ready to tell the caller the earliest ready time.
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

> n8n calculates the minimum ready time entirely server-side. No time or date needs to be sent from GHL.

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
// Returns minimum_time: earliest pickup = now + 30 min, rounded up to next :00 or :30

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

function roundUpToSlot(minutes) {
  const remainder = minutes % 30;
  return remainder === 0 ? minutes : minutes + (30 - remainder);
}

function minutesToTime(minutes) {
  const m = ((minutes % 1440) + 1440) % 1440;
  const h = Math.floor(m / 60);
  const min = m % 60;
  const period = h >= 12 ? 'PM' : 'AM';
  const display = h === 0 ? 12 : h > 12 ? h - 12 : h;
  return `${display}:${min.toString().padStart(2, '0')} ${period}`;
}

function isOpen(minutes) {
  const m = ((minutes % 1440) + 1440) % 1440;
  return (m >= 720 && m <= 1439) || (m >= 0 && m < 120);
}

const DEFAULT_OPEN_TIME = 750; // 12:30 PM — first valid slot when restaurant opens

let minimumRaw;
let status;

const isClosed = currentTotal >= 120 && currentTotal < 720;

if (isClosed) {
  // Restaurant not yet open — minimum is 12:30 PM (first slot after opening)
  minimumRaw = DEFAULT_OPEN_TIME;
  status = currentTotal < 720 ? 'before_opening' : 'after_closing';
} else {
  const earliest = roundUpToSlot(currentTotal + 30);
  if (isOpen(earliest)) {
    minimumRaw = earliest;
    status = 'open';
  } else {
    // Near closing — no valid slot before 2:00 AM, push to next opening
    minimumRaw = DEFAULT_OPEN_TIME;
    status = 'near_closing';
  }
}

return [{
  json: {
    status,
    minimum_time: minutesToTime(minimumRaw)
  }
}];
```

---

## Step 6 — n8n Output

```json
{
  "status": "open",
  "minimum_time": "2:00 PM"
}
```

**Status values:**

| Status | Meaning | minimum_time returned |
|--------|---------|----------------------|
| `open` | Restaurant is open, calculated from now | now + 30 min, rounded to next :00 or :30 |
| `before_opening` | Call before 12:00 PM | 12:30 PM |
| `after_closing` | Call after 2:00 AM | 12:30 PM |
| `near_closing` | Open but no valid slot before 2:00 AM | 12:30 PM |

---

## Step 7 — How Sofia Uses the Response

In GHL, map the action response variable:
- `minimum_time` → variable Sofia references as the earliest ready time

Sofia says (STEP 7 of order flow):
> "The earliest we can have that ready is [minimum_time] — what time works for you?"

The caller's answer (e.g., "2:30 PM", "in an hour", "as soon as possible") becomes `requested_time`.
If the caller asks for a time earlier than `minimum_time`, Sofia says: "The earliest we can do is [minimum_time] — does that work?"

---

## Example Scenarios

| Edmonton Time | Status | minimum_time |
|---------------|--------|--------------|
| 11:35 AM | `before_opening` | 12:30 PM |
| 1:45 PM | `open` | 2:30 PM |
| 1:15 AM | `near_closing` | 12:30 PM (next opening) |
| 2:30 AM | `after_closing` | 12:30 PM |

---

*End of Get Ready Time Action — The Greek Grill AB*
