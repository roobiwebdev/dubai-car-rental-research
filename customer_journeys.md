# The Dubai Rental Customer Journey — Evidence-Based

Everything in this document is reconstructed from operator T&Cs, FAQs, government sources and
customer-complaint evidence. Design recommendations are labelled `INFERRED / RECOMMENDED`.

---

## Stage 1 — Initial inquiry

### What customers actually ask
Derived from operator FAQ pages, published contact instructions, and marketplace listing behaviour:

| Intent | Typical phrasing | Why it matters |
|---|---|---|
| Availability + price | "How much is a BMW for 3 days?" | The dominant opener |
| Urgency | "I need a car tomorrow" / "today, now" | Delivery SLA is the differentiator (30–60 min claims) |
| Deposit anxiety | "Can I rent without a deposit?" | Zero-deposit is a whole market segment |
| Delivery | "Can you deliver to DXB?" / "to my hotel in Marina?" | Free delivery is near-universal; airports are special-cased |
| Eligibility | "I'm a tourist, what documents do I need?" | Resident vs tourist rules genuinely differ |
| Licence validity | "My licence is from India/UK/Russia — is it OK?" | Reciprocal-agreement rules; residents *cannot* use an international licence |
| Mileage | "Is there a km limit?" | 250 km/day is the market norm; "unlimited" is a selling point |
| Price anchor | "OneClickDrive says AED 89/day" | Marketplace-originated leads arrive pre-priced |
| Specific car from social | "The white G63 from your Instagram" | Instagram drives ~41% of premium traffic |

### `INFERRED / RECOMMENDED` — agent behaviour at this stage
- Answer the **asked** question first, then ask for at most **two** missing slots.
- Never open with a menu wall. The market's whole advantage is that WhatsApp feels human.
- If the customer named a price anchor, **check it against the backend before agreeing or denying**,
  and say plainly which one is current.

---

## Stage 2 — Requirement collection

### The minimal slot set (matches operators' own published instructions)
**Required to quote:**

| Field | Required? | Notes |
|---|---|---|
| Vehicle or category | **Yes** | Category is enough to quote; specific unit needed to confirm |
| Start date + time | **Yes** | Time matters — delivery slots and buffer logic depend on it |
| End date + time | **Yes** | Duration drives daily/weekly/monthly rate tiering |
| Delivery / pickup location | **Yes** | Determines delivery fee and driver routing |
| Residency status (resident / tourist / GCC) | **Yes** | Determines the entire document set |
| Driver age | **Yes** | Age gates whole vehicle classes (see below) |

**Required to confirm the booking:**

| Field | Required? | Notes |
|---|---|---|
| Full name (as on licence) | Yes | Must match documents and the RTA contract |
| Phone | Yes (implicit from WhatsApp) | Confirm it is the driver's number |
| Nationality / licence-issuing country | Yes | Determines whether an IDP is required |
| Driving licence | Yes | Front and back (Rentico requires both) |
| Passport (+ visa page / entry stamp) | Tourists: Yes | Panorama explicitly requires the airport entry stamp |
| Emirates ID | Residents: Yes | |
| Payment method | Yes | Card for pre-auth, or agreed cash-on-delivery |
| Drop-off location | If different | |

**Optional / upsell:**
Additional driver · CDW / insurance upgrade · extra mileage package · child seat · chauffeur ·
airport meet-and-greet · fuel option · Salik package.

### Age rules found (they vary widely — must be data, not prompt text)
| Operator | Rule |
|---|---|
| Rentico | 18+; ages 18–21 require an **AED 2,000** deposit |
| BE VIP | 18 minimum |
| Octane | states 18 minimum; notes most companies require **25 with 3 years' experience** |
| Dubai Rent A Car | economy **20+** with 1 year experience; luxury **23+** |
| Saadatrent | **21+**; luxury **25+** |
| Luxury Supercars Dubai | **21–65**, licence held ≥1 year |
| Market/press general | minimum 21 commonly cited |

> `INFERRED / RECOMMENDED` Age eligibility must be a **per-vehicle-class rule in the database**, not
> a sentence in the system prompt. It changes by class *and* by insurance policy.

---

## Stage 3 — Availability

### How the market really manages it
- **Systems companies** (Hertz, Europcar, Enterprise) run real reservation systems with live inventory.
- **WhatsApp companies** — the majority — check manually: a spreadsheet, a fleet WhatsApp group, or
  a staff member's memory. Some run a rental ERP (Caryaati, Appic Fleet, HQ Rental) behind the scenes.
- Marketplaces (OneClickDrive) show **listings, not inventory** — the listing is an advert, and
  availability is confirmed only in the WhatsApp conversation that follows.

### The availability model the agent needs

**Two-level availability:**
1. **Category level** — "do we have *a* mid-size SUV for these dates?" → used for recommendations
   and soft quotes.
2. **Unit level (VIN/plate)** — "is *this specific* Urus free?" → required before any confirmation.

**Factors that must reduce availability:**

| Factor | Why | Evidence |
|---|---|---|
| Existing reservations | Obvious | Universal |
| **Turnaround buffer** | Cleaning, inspection, refuelling between rentals | Panorama: cars delivered "clean, fuelled and ready" |
| **Delivery / collection travel time** | The car is unavailable while in transit | NCK: 1–3h delivery; Superior: 60 min; Panorama: 30 min |
| Scheduled maintenance / service | Fleet ops | Caryaati/HQ Rental both model maintenance schedules |
| **Mulkiya (registration) renewal** | UAE-specific vehicle downtime | Appic Fleet automates this |
| Damage / accident repair | Post-incident | Insurance replacement typically 7–15 days |
| **Late returns from the current renter** | Extremely common | 1-hour grace is standard; 3+ hours late = a full day charged |
| Extension by the current renter | Consumes the next customer's slot | Luxury Supercars Dubai documents exactly this conflict |

### `INFERRED / RECOMMENDED` — the hard rule

> **The AI must never assert availability. It may only report what `check_availability()` returned,
> in the same turn.**

Implementation:
1. The model **cannot** answer an availability question without calling the tool.
2. The tool returns unit IDs, an availability window, **and a `quote_token`**.
3. `create_booking()` re-validates atomically against the same source and **fails closed** on conflict.
4. Between quote and confirmation, place a **soft hold** (recommended TTL: **30 minutes**, longer for
   rare/exotic units) and state the expiry to the customer in the message.
5. If the hold expires or the unit is taken, the agent **must** re-check before confirming and offer
   substitutes — never silently confirm.

**Double-booking prevention is a database constraint, not a prompt instruction.** Use an exclusion
constraint / serialisable transaction on `(vehicle_id, time_range)` including buffers.

---

## Stage 4 — Pricing

### How Dubai operators structure price (verified)

| Component | Market practice | Evidence |
|---|---|---|
| **Daily rate** | Headline price; economy from ~AED 39–99/day, luxury AED 300–5,499/day, supercars to AED 10,500/day | Saadatrent, Star Empire, Superior |
| **Weekly rate** | Discounted vs daily | Universal |
| **Monthly rate** | Sharply discounted; from ~AED 999–1,200/month | Quick Lease, Dubai Rent A Car |
| **Long-term/lease** | Contract pricing, often with lease-to-own | Quick Lease, SelfDrive, eZhire |
| **Seasonal / weekend uplift** | Practised; rarely published | Inferred from demand patterns; treat as backend rule |
| **VAT** | **5%**, sometimes included in the quoted price | Dubai Rent A Car states 5% VAT included |
| **Deposit** | AED 1,000–10,000 by class; or **pre-auth AED 2,000**; or zero-deposit for a fee | Dubai Rent A Car, Superior, Rentico |
| **CDW / insurance** | Basic third-party included; comprehensive/CDW optional; **deductible AED 1,000–1,500** | Rentico, Superior, NCK |
| **Mileage** | **250 km/day** typical (200–300 range); unlimited on some plans | Rentico, BE VIP, Panorama, Star Empire, Renty |
| **Excess mileage** | **AED 1–5 per km** (market reports 0.50–1.00 common) | Rentico publishes AED 1–5 |
| **Delivery** | Often free; otherwise **AED 99 Dubai / AED 99 DXB / AED 149 DWC**; free above AED 1,000 | Rentico |
| **Airport** | Airport delivery common; **customer pays airport parking fee**; counter suppliers add an airport fee | Renty; DXB T1/T3 parking AED 10 + AED 5/30min |
| **Additional driver** | Fee, must be named on contract and insurance | Superior offers additional driver cover |
| **Salik tolls** | **AED 4/gate + admin**, billed as **AED 5–6/crossing**; BE VIP publishes AED 5 | BE VIP, market reporting |
| **Traffic fines** | Passed through **+ ~AED 50 admin per ticket** | Market reporting |
| **Cleaning** | Charged if returned excessively dirty | Common T&C |
| **Fuel** | Return at same level; refuelling charged at a premium | BE VIP, Panorama |
| **Late return** | **1-hour grace**, then **AED 20–80/hour** (some report AED 50–200); **3+ hours = full day** | Luxury Supercars Dubai; market reporting |
| **Cancellation** | Varies; **48-hour notice** for luxury modifications/cancellations | Luxury Supercars Dubai |

### `INFERRED / RECOMMENDED` — the hard rule

> **Every number the agent states must come from `get_quote()`. The model performs no arithmetic on
> money.** Not the daily total, not the VAT, not the mileage overage estimate. It renders the
> backend's itemised breakdown.

The quote object should carry: line items, subtotal, VAT, deposit/pre-auth amount and type, mileage
cap and overage rate, delivery fee, what is *not* included (Salik, fines, fuel), the
**quote ID**, and an **expiry timestamp**.

**Why this is non-negotiable:** the documented market failure is *"charges confirmed via WhatsApp
showed different amounts on the invoice."* A model that free-hands a price recreates that failure at
machine speed.

---

## Stage 5 — Documents

### Requirements by customer type (verified across operators)

| | **UAE Resident** | **Tourist / Visitor** | **GCC National** |
|---|---|---|---|
| Driving licence | **UAE licence, mandatory** — an international licence is **not accepted** for residents (Modena) | Home-country licence | GCC licence generally accepted |
| IDP | Not applicable | **Required if the home country has no reciprocal agreement** with the UAE | Generally not required |
| ID | **Emirates ID** (unexpired) | Passport, valid for the full rental period | Passport / GCC ID |
| Visa | — | Tourist/visit visa + **entry stamp** (Panorama explicitly) | — |
| Card | Credit card in the renter's name commonly required for pre-auth | Same; some accept debit/cash | Same |
| Extra | Proof of residency (lease/utility bill) sometimes for long rentals (Modena) | — | — |

### Where document collection should happen

`INFERRED / RECOMMENDED` — **do not let WhatsApp be the system of record.**

```
Customer sends photo in WhatsApp
        ↓
Webhook downloads media immediately (Meta media URLs are short-lived)
        ↓
Stored ONLY in encrypted object storage (private bucket, server-side encryption)
        ↓
Document record created: type, expiry, status=pending, booking_id, hash
        ↓
OCR / automated checks (name match, expiry date, licence class)
        ↓
HUMAN REVIEW for accept/reject   ← required in MVP
        ↓
Booking moves to documents_verified
        ↓
Media reference in the conversation log is a POINTER, never the image
```

Rules:
- The AI may **request, receive, acknowledge and classify** documents. It may **not** be the party
  that approves them in MVP — a rejected-document decision has legal and insurance consequences.
- **Never** echo document contents (passport number, ID number) back into the chat.
- Retention: keep only as long as the rental + statutory record-keeping requires, then delete. UAE
  **PDPL (Federal Decree-Law 45/2021)** requires data not be kept longer than necessary, with
  explicit purpose-limited consent.
- Compare against `eZhire`'s model: national ID/passport + licence + card copy, uploaded from the
  phone, verified before delivery. That is the target UX — we are just doing it in WhatsApp instead
  of an app.

---

## Stage 6 — Payment, deposit and booking confirmation

### What the market does
- **Pay on delivery** is normal and even a selling point (Dubai Rent A Car: "no pre-payment required";
  Rentico: "you pay the full rental amount when you receive the car").
- **Pre-authorisation instead of a cash deposit** is the cleanest model (Superior: standard
  **AED 2,000** hold, released after return and inspection, *never charged unless damage occurs*).
- **Zero-deposit** is a full market segment. How it actually works: the operator carries the risk on
  selected cars, or takes a smaller **fines hold (~AED 700–1,200)** returned in ~20 days, or charges a
  **small no-deposit fee (from ~AED 27)**. Liability for fines, tolls and damage above the excess
  **still sits with the renter** — just billed afterwards.
- **Deposit refund timing** is the market's biggest trust problem: 7 days (Saadatrent) / 21 days
  (Dubai Rent A Car) / 28 days (Renty) / 30 working days (BE VIP). The delay is real — traffic fines
  and Salik take 2–3 weeks to post.
- **The legal backstop:** Dubai's Consumer Protection authority (DET) requires card holds and
  deposits to be **returned within 30 days** of vehicle return, refunded in cash or by bank transfer
  if paid that way, **with the company covering transaction fees**.
- Payment methods observed: Visa/Mastercard/Amex, Apple Pay, cash, bank transfer, **Tabby (BNPL)**,
  **crypto (USDT/Bitcoin)**, and **Russian cards**.

### `INFERRED / RECOMMENDED` — responsibility split

| Action | Who |
|---|---|
| Explain deposit type and amount | **AI** (reading from the quote) |
| Generate and send a **payment link** | **AI → payment provider** (link generated by backend, never fabricated) |
| Confirm payment received | **Payment provider webhook only.** The AI never infers success from "I paid" |
| Create the pre-authorisation | Backend + provider |
| Release the pre-auth / refund the deposit | **Human approval** (finance), AI reports status |
| Issue a refund | **Human approval** — always |
| Take cash on delivery | Delivery driver, recorded in the backend |

> **Never** say "payment received" without a provider confirmation event. Say *"I've sent the payment
> link — I'll confirm the moment it clears."*

### Booking confirmation
`INFERRED / RECOMMENDED` — confirmation must be a **backend-generated artefact**, not a chat message:
booking ID, vehicle (make/model/plate if allocated), start/end datetime, delivery address, itemised
price, deposit terms, mileage cap, and the cancellation/extension policy. Send as a WhatsApp **utility
template** plus a PDF. In Dubai this ties directly into the **RTA TARS unified rental contract** —
digital signature, documented vehicle condition, clear obligations (see below).

---

## Stage 7 — Delivery and pickup

### The Dubai-specific reality

| Location | Practice |
|---|---|
| **DXB (T1/T2/T3)** | Counter operators inside arrivals; delivery-based operators use **meet-and-greet** at the terminal exit. Meet-and-greet is often cheaper because it avoids the airport concession fee. **Customer usually pays airport parking** (T1/T3: AED 10 + AED 5/30min; Car Park B AED 100/day) |
| **DWC (Al Maktoum)** | Served, typically at a higher delivery fee (Rentico: AED 149 vs AED 99 for DXB) |
| **Hotels** | Free delivery is near-universal for luxury; concierge handover common |
| **Residences / Marina, Downtown, Palm, Business Bay, Al Barsha, JBR, JLT** | The standard free-delivery zone list (NCK publishes exactly this set) |
| **Offices** | Common for corporate/monthly |
| **Other emirates** | Extra fee; delivery/collection outside city limits quoted higher (one airport operator: AED 52.50 in-city vs AED 157.50 outside) |

**Delivery SLAs claimed:** Panorama **30 minutes**; Superior and Star Empire **60 minutes**;
NCK **1–3 hours**, express 1 hour; eZhire **from 10 minutes**, typically ~1 hour.

### What should happen over WhatsApp
`INFERRED / RECOMMENDED`

1. **Capture the location as a WhatsApp location pin**, not a typed address. Dubai addressing is
   unreliable; a pin is unambiguous and routes the driver directly.
2. Confirm a **delivery window**, not a promise of an exact minute.
3. On dispatch, send a **utility template**: driver name, phone, vehicle, plate, ETA.
4. **Handover checklist in-thread:** photos of the vehicle condition (all four sides + odometer +
   fuel gauge), signed contract, keys. Under **RTA TARS** the unified rental contract already
   requires digital signature and documented vehicle condition — align our handover artefact to it.
5. **Return:** reminder template 24h and 2h before; return location confirm; return inspection photos;
   deposit status message.

> The handover photos are the client's single best defence against damage disputes — the #2 complaint
> category after deposits. Make them mandatory, timestamped, and attached to the booking.

---

## Stage 8 — After booking (the neglected half of the market)

**No Dubai operator we researched automates this. It is our largest differentiation surface.**

| Event | Verified market practice | `INFERRED / RECOMMENDED` agent behaviour |
|---|---|---|
| **Confirmation** | Free-text in chat | Backend artefact + PDF + utility template |
| **Pre-delivery reminder** | Ad hoc | Template with driver + ETA |
| **Extension** | Extensions are **minimum 1 day** — no hourly extensions in this market. Depends on availability | Check availability → quote the difference → payment link → **confirm only after payment**. If the unit is taken, **offer a substitute** (Luxury Supercars Dubai's exact policy) |
| **Upgrade** | Offered informally | Availability check + price difference + human approval above a threshold |
| **Additional driver** | Must be named and covered | Collect documents → human verification → add to contract |
| **Early return** | Varies; often no automatic refund | Never promise a refund — quote policy, escalate for goodwill |
| **Late return** | **1-hour grace**, then hourly (AED 20–80, some report 50–200); **3+ hours = a full day**. Critically: **insurance may be void after the contracted return time** | Proactive message at grace-period start. **Always mention the insurance risk** — it is a safety issue, not just a fee |
| **Accident** | Mandatory: call **999**; do not move the vehicle unless police direct or damage is minor and no injuries; **police report is mandatory** for insurance; report obtainable within ~7 days; minor no-injury accidents can be filed in the **Dubai Police app**; renter pays the excess | **Safety script first, then immediate human escalation.** See flow in `agent_design.md` |
| **Breakdown** | 24/7 roadside assistance standard (Hertz: with all packages; Enterprise: free, number on the windscreen sticker). Covers mechanical help, towing, **replacement car**, lockouts, lost keys. Not valid outside the UAE | Collect vehicle + location pin + symptom → dispatch → human ownership |
| **Flat tyre** | **Tyre and rim damage is excluded from standard cover AND CDW** (Superior) | Never tell the customer it is covered. Quote policy, escalate |
| **Lost key** | Covered under roadside assistance by some | Escalate — charges vary hugely |
| **Salik** | AED 4/gate + admin → billed AED 5–6; charged at the end | Proactive running total on request; itemised at return |
| **Traffic fine** | Attached to the plate, matched to the contract, charged to the card on file **+ ~AED 50 admin** | Notify with evidence (date, location, amount, admin fee). Never dispute on the customer's behalf without a human |
| **Damage dispute** | Highest-heat category | Immediate human handoff with handover photos attached |
| **Deposit refund** | 7–30 days; legally **within 30 days** (Dubai Consumer Protection) | **Proactive status updates.** This alone would beat the entire market |

---

## Stage 9 — Language

### Evidence for prioritisation

| Signal | Data |
|---|---|
| Dubai visitors 2025 | **19.59m** total |
| Largest single country market | **India — 2.6m** |
| CIS & Eastern Europe | **2.89m (15%)** — Russia alone ~2.58m arrivals |
| Western Europe | **4.1m (21%)** |
| GCC | **2.99m (15%)**; wider MENA **2.17m (11%)** |
| South Asia | **2.89m (15%)** |
| Operator language sets | Renty: 10 languages. Rentico: **EN/RU/AR**. BE VIP: **AR/EN**. SelfDrive SIA: **40+** |
| Resident population | Large Indian, Pakistani, Filipino, Arab expatriate communities |

### `INFERRED / RECOMMENDED` — MVP language priority

| Tier | Languages | Rationale |
|---|---|---|
| **MVP (day one)** | **English, Arabic, Russian** | English is the market lingua franca; Arabic is required for GCC/local customers and is the national language; Russian is the single highest-ROI non-English language given 2.89m CIS visitors and the fact that two shortlisted competitors specifically target it (Rentico's Russian cards, Telegram channels) |
| **V1** | **Hindi/Urdu** | 2.6m Indian visitors + the largest resident expatriate group. Treat Hindi/Urdu as one conversational tier |
| **V2** | French, German, Chinese | Western Europe volume + growing NE/SE Asia (1.85m) |

### How to handle language properly (not just "translate")
1. **Detect from the first message**, store on the Customer record, and persist across sessions.
2. **Allow explicit switching** at any time ("عربي", "по-русски") without losing slot state — language
   is a rendering property of the conversation, not a new conversation.
3. **Policy text must be pre-translated and human-approved.** Do **not** let the model translate
   insurance, liability or deposit terms on the fly. Store approved translations per policy clause;
   the model selects, it does not compose.
4. **Numbers, dates and money always render identically** across languages, in AED, 24h format,
   Gulf Standard Time.
5. **Handoff must be language-aware** — route to a human who speaks the customer's language, and put
   the language in the handoff payload.
6. Voice notes are common in Arabic/Hindi/Urdu — **transcribe, then process**.

---

## Stage 10 — Edge cases and correct behaviour

| # | Situation | Correct behaviour |
|---|---|---|
| 1 | Requested car unavailable | State it plainly, offer **2–3 backend-verified alternatives** in the same or adjacent class with prices. Never "let me check and get back to you" without a follow-up commitment |
| 2 | Specific model unavailable for the dates | Offer nearest dates it *is* free, plus same-class substitutes |
| 3 | Booking overlaps another | Backend rejects; agent apologises, re-quotes options. Never override |
| 4 | Vehicle becomes unavailable after quoting | **Proactive** message, apology, substitutes, and a discount/upgrade offer flagged for human approval |
| 5 | Customer wants cheaper | Offer a lower class or longer-duration rate tier. **Never invent a discount** — only apply promo codes/rules that exist in the backend |
| 6 | Changes dates | Re-run availability + re-quote. Old quote is void; say so |
| 7 | Changes location | Re-quote delivery fee (airport vs city differ) |
| 8 | Wants an additional driver | Collect documents, human verification, contract amendment, fee |
| 9 | Wants an extension | Availability → price difference → payment → confirm. Minimum 1 day. If blocked, offer substitution |
| 10 | Running late | Explain grace period + hourly charge + **insurance risk**; offer to convert to a formal extension |
| 11 | Wants early return | Quote policy; **never promise a refund**; escalate goodwill decisions |
| 12 | Lost key | Escalate to human + roadside; do not quote a replacement cost from memory |
| 13 | **Accident** | Safety script → 999 → do not move if serious → immediate human + roadside escalation. **Highest priority path** |
| 14 | Breakdown | Location pin + symptom → roadside dispatch → human owns it |
| 15 | Damage found at return | Do not adjudicate. Attach handover photos, escalate to human |
| 16 | Payment failure | Never assume; re-issue link, offer alternative method, escalate after 2 failures |
| 17 | Deposit dispute | **Immediate human escalation.** Provide factual status and the legal 30-day framework — nothing more |
| 18 | Document rejected | Explain the specific reason (blurred / expired / name mismatch), request a re-upload. Rejection decision itself is human-made in MVP |
| 19 | Customer goes silent | One follow-up at +2h (in-window), one utility template at +24h, then release the soft hold and tell them it was released |
| 20 | Voice message | Transcribe → confirm understanding in one line → proceed |
| 21 | Image sent | Classify: document / damage photo / car-they-want (e.g. an Instagram screenshot) / irrelevant. Route accordingly |
| 22 | Location pin sent | Reverse-geocode, confirm the area name back, use for delivery quote |
| 23 | Multiple requests in one message | Acknowledge **all**, answer in order, do not silently drop one |
| 24 | Another language mid-conversation | Switch, retain all slots, confirm the switch in one line |
| 25 | Outside knowledge base | Say you do not know, offer a human. **Never guess a policy** |
| 26 | Asks for something illegal/unsafe (drive to another country, unlicensed driver, fake documents) | Refuse clearly, cite policy, escalate if pressed |
| 27 | Angry / abusive | De-escalate once, then hand off. Do not argue |
| 28 | Duplicate/parallel enquiries from the same number | Merge into one conversation thread; do not create two bookings |
| 29 | Wrong number / spam | Polite close; do not consume agent resources |
| 30 | Marketplace price anchor conflicts with live price | State the current backend price and the reason (dates/season/class), and offer the nearest option at the anchored price if one exists |
