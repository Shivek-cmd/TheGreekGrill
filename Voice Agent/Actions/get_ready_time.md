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
// Restaurant hours:
// Mon–Thu: 9:00 AM – 10:00 PM  (OPEN_TIME=540, CLOSE_TIME=1320)
// Fri–Sun: 9:00 AM – 12:00 AM  (OPEN_TIME=540, CLOSE_TIME=1440)
// Returns minimum_time: earliest pickup = now + 30 min, rounded to next :00 or :30

  const now = new Date();

  const parts = new Intl.DateTimeFormat('en-CA', {
    timeZone: 'America/Edmonton',
    weekday: 'long',
    hour:    'numeric',
    minute:  '2-digit',
    hour12:  false
  }).formatToParts(now);

  const dayOfWeek    = parts.find(p => p.type === 'weekday').value;
  const currentHour  = parseInt(parts.find(p => p.type === 'hour').value);
  const currentMinute = parseInt(parts.find(p => p.type === 'minute').value);
  const currentTotal = currentHour * 60 + currentMinute;

  const OPEN_TIME  = 540;  // 9:00 AM every day
  const CLOSE_TIME = ['Friday', 'Saturday', 'Sunday'].includes(dayOfWeek) ? 1440 : 1320;
  // Fri/Sat/Sun → midnight (1440 = end of day) | Mon–Thu → 10:00 PM (1320)

  const DEFAULT_OPEN_SLOT = 570; // 9:30 AM — first ready slot after opening

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

  const isCurrentlyOpen = currentTotal >= OPEN_TIME && currentTotal < CLOSE_TIME;

  let minimumRaw;
  let status;
  let is_open;

  if (!isCurrentlyOpen) {
    minimumRaw = DEFAULT_OPEN_SLOT;
    is_open    = false;
    status     = currentTotal < OPEN_TIME ? 'before_opening' : 'after_closing';
  } else {
    const earliest = roundUpToSlot(currentTotal + 30);
    if (earliest < CLOSE_TIME) {
      minimumRaw = earliest;
      is_open    = true;
      status     = 'open';
    } else {
      // Near closing — no valid slot before close, push to next opening
      minimumRaw = DEFAULT_OPEN_SLOT;
      is_open    = false;
      status     = 'near_closing';
    }
  }

  return [{
    json: {
      status,
      is_open,
      minimum_time: minutesToTime(minimumRaw)
    }
  }];
```

---

## Step 6 — n8n Output

```json
// Restaurant is open:
{ "status": "open", "is_open": true, "minimum_time": "10:30 AM" }

// Restaurant is closed:
{ "status": "after_closing", "is_open": false, "minimum_time": "9:30 AM" }
```

**Status values:**

| Status | is_open | Meaning | minimum_time |
|--------|---------|---------|--------------|
| `open` | true | Restaurant is open, calculated from now | now + 30 min, rounded to next :00 or :30 |
| `before_opening` | false | Called before 9:00 AM | 9:30 AM |
| `after_closing` | false | Called after closing time | 9:30 AM (next opening) |
| `near_closing` | false | Open but no valid slot before closing | 9:30 AM (next opening) |

---

## Step 7 — How Sofia Uses the Response

In GHL, map these response variables:
- `is_open` → Sofia checks this first to decide what to say
- `minimum_time` → earliest ready time (used in both open and closed scenarios)

**When `is_open` is true:**
> "The earliest we can have that ready is [minimum_time] — what time works for you?"

Caller's answer becomes `requested_time`. If they ask for an earlier time: "The earliest we can do is [minimum_time] — does that work?"

**When `is_open` is false:**
- `before_opening`: "We're not open just yet — we open at 9 AM. I can take your order now and have it ready from [minimum_time] — want to do that?"
- `after_closing` / `near_closing`: "We're actually closed for tonight — we open again tomorrow at 9 AM. I can take your order now for pickup or delivery tomorrow from [minimum_time] — want to do that?"
- If caller says yes → continue, `requested_time` = [minimum_time]
- If caller says no → "No problem at all — give us a call when you're ready. Have a great night!" and end the call
- Dine-in is not available when closed — if caller wants dine-in, explain and offer takeout/delivery for next opening instead

---

## Example Scenarios

| Day | Edmonton Time | Status | is_open | minimum_time |
|-----|---------------|--------|---------|--------------|
| Monday | 8:45 AM | `before_opening` | false | 9:30 AM |
| Monday | 10:30 AM | `open` | true | 11:00 AM |
| Monday | 9:35 PM | `near_closing` | false | 9:30 AM (tomorrow) |
| Monday | 10:30 PM | `after_closing` | false | 9:30 AM (tomorrow) |
| Friday | 11:45 PM | `near_closing` | false | 9:30 AM (tomorrow) |
| Saturday | 2:00 PM | `open` | true | 2:30 PM |

---

*End of Get Ready Time Action — The Greek Grill AB*
