# The Greek Grill AB — Technical Settings

> Configure these in GHL Voice AI → Agent → Settings tabs. Every value below is tuned for a restaurant ordering call: short calls, Greek and international food names, English-speaking callers, natural pacing.

---

## Latency Fix Guide

> **If you're experiencing 3–4 second delays**, work through this list in order — each fix targets a different layer.

| Layer | Cause | Fix |
|-------|-------|-----|
| **STT** | Accurate transcription model adds ~1–2s per turn | Switch to Fast + rely on boosted keywords |
| **LLM** | Long prompt = more tokens = slower inference | Already optimised — keep prompt under 900 words |
| **Webhook** | n8n cold start or slow processing | Respond immediately, do heavy work after |
| **TTS** | Long AI responses = more audio to synthesise | Shorter sentences in prompt |
| **GHL** | Wait Before Speaking set too high | Reduce to 0.1s |
| **Response Speed** | Set too low | Increase to 0.8 |

**Quick wins (do these first):**
1. Change transcription model: Accurate → **Fast**
2. Change Wait Before Speaking: 0.5s → **0.1s**
3. Change Response Speed: 0.65 → **0.8**
4. In n8n: respond with `200 OK` immediately, process order asynchronously

---

---

## 1. Call Settings

| Setting | Value | Reason |
|---------|-------|--------|
| **Max Call Time** | `8 minutes` | Restaurant calls rarely exceed 5 min. 8 min gives buffer for long orders or chatty callers. |
| **End Call After Silence** | `15 seconds` | If caller goes silent for 15s after all reminders, end the call gracefully. |
| **Wait Before Speaking** | `0.1 seconds` | Reduced from 0.5s — major latency source. 0.1s still sounds natural, 0.5s adds dead air every single turn. |

### Idle Time Reminder
| Setting | Value |
|---------|-------|
| **Enable** | ON |
| **When Reminder Starts** | `5 seconds` of silence |
| **How Often They Repeat** | `3 times` |

**What to say on reminder:** Configure the reminder message as:
> "Hey, are you still there?"
> (second reminder) "Just checking — did you want to continue?"
> (third/final) "I'll go ahead and end the call if you're done. Thanks for calling The Greek Grill!"

---

## 2. Agent Behaviour

| Setting | Value | Reason |
|---------|-------|--------|
| **Response Speed** | `0.8` | Increased from 0.65 — faster response generation. Restaurant callers expect quick replies; lower values add noticeable delay. |
| **Interruption Sensitivity** | `0.60` | Mid-range — lets callers cut in naturally (they often jump in while the agent is speaking) without triggering on background noise. |
| **LLM Temperature** | `0.3` | Low temperature = consistent, predictable responses. Critical for order-taking — you don't want creative variations when confirming an order. |

### Backchannel
| Setting | Value |
|---------|-------|
| **Enable Backchannel** | ON |
| **Backchannel Frequency** | `0.4` |
| **Backchannel Word List** | `Got it`, `Sure`, `Okay`, `Mhm`, `Yeah`, `Right`, `Yep` |

> Backchannel makes Sofia sound like she's actively listening while the caller speaks. Keep frequency at 0.4 — too high sounds annoying.

---

## 3. Transcription & Speech

### Transcription Model
**Select: Fast** *(changed from Accurate for latency)*

> Accurate adds 1–2 seconds per turn — too much for a live phone call. Switch to Fast and compensate with boosted keywords and pronunciation dictionary (already configured below). Greek and Italian food names are the most common STT failure points — all are covered in the boosted keyword list. If a specific item still gets misheard after switching, add it as an individual boosted keyword entry. If Fast is still unacceptable for accuracy, switch back to Accurate and accept the latency tradeoff.

### Vocabulary Specialisation
**Select: General**

*(Medical is not relevant. General handles English-speaking callers with Greek and international food names.)*

### Boosted Keywords
Add all menu item names to improve recognition accuracy. Paste these exactly:

```
Tzatziki, Souvlaki, Skepasti, Calamari, Arancini, Bruschetta, Focaccia,
Rack of Lamb, Butter Chicken, Blackened Salmon, Wagyu, Karaage, Piccata,
Carbonara, Bolognese, Risotto, Provencal, Chimichurri, Szechuan,
Margherita, Pinocchio, Cappuccino, Espresso, Latte, Clamato,
Greek Grill, Sofia, mild, medium, spicy, takeout, reservation, dine-in
```

> **Known STT risk — "Tzatziki" is commonly misheard**, often transcribed as "satsiki" or "zatsiki". The boosted keyword `Tzatziki` above directly targets this. If it still happens, also add `Tsatziki` as a variant boosted keyword. The pronunciation dictionary entry `Tzatziki → tsah-tsee-kee` also ensures Sofia says it correctly so callers recognise it when it's suggested.

### Pronunciation Dictionary
Add each entry as a separate word + pronunciation pair:

| Word | Pronunciation |
|------|--------------|
| Tzatziki | tsah-tsee-kee |
| Souvlaki | soo-vlah-kee |
| Skepasti | skeh-pahs-tee |
| Chimichurri | chim-ih-chur-ee |
| Arancini | ah-ran-chee-nee |
| Focaccia | foh-kah-chuh |
| Bruschetta | broo-sket-uh |
| Bolognese | boh-loh-nyeh-zeh |
| Carbonara | kar-buh-nar-uh |
| Risotto | rih-zot-oh |
| Piccata | pih-kah-tuh |
| Wagyu | wag-yoo |
| Karaage | kah-rah-ah-geh |
| Calamari | kah-luh-mar-ee |
| Provencal | proh-vahn-sahl |
| Cappuccino | kap-uh-chee-noh |
| Espresso | eh-spres-oh |
| Pinocchio | pih-noh-kee-oh |
| Clamato | kluh-may-toh |
| Provolone | proh-voh-loh-neh |
| Penne | pen-ay |
| Arugula | ah-roo-gyoo-luh |
| Balsamic | ball-sam-ik |
| Basmati | baz-mah-tee |
| Cremini | kreh-mee-nee |
| Zucchini | zoo-kee-nee |
| Truffle | truf-ul |
| Fennel | fen-ul |
| Feta | fee-tuh |

---

## 4. Voice Settings

### Noise Cancellation
**Select: Remove Noise**

> Callers may be in kitchens, cars, or noisy environments. "Remove Noise" handles this without stripping the caller's voice like "Remove Noise + Background Speech" can.

### Background Sound
**Select: None**

> Don't add artificial background sounds. This is a restaurant call, not a coffee shop chatbot. A clean line sounds more professional.

### Voice Levels

| Setting | Value | Reason |
|---------|-------|--------|
| **Voice Speed** | `1.0` | Default natural speed. Let Dynamic Voice Speed handle adjustments. |
| **Voice Temperature** | `0.65` | Slightly expressive — Sofia is warm and human, not flat. Don't go above 0.8 or responses sound inconsistent. |
| **Voice Volume** | `1.0` | Default. Adjust only if testers report Sofia sounding too quiet or loud. |

### Toggles

| Setting | Value | Reason |
|---------|-------|--------|
| **Speech Normalization** | ON | Converts `$38.95` → "thirty-eight ninety-five", `7:15 PM` → "seven fifteen PM". Essential for order recaps with prices and times. |
| **Dynamic Voice Speed** | ON | Automatically slows down if the caller speaks slowly (e.g., elderly callers or callers unfamiliar with the menu). Feels much more natural. |

---

*End of Technical Settings — The Greek Grill AB | Sofia*
