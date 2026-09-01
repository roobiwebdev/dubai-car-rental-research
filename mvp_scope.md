# Scope — MVP / V1 / V2 / Future

Principle: **the MVP is the smallest thing that beats a human replying at 2am, without ever lying to
a customer.** Everything that cannot be made truthful is deferred, not shipped half-done.

---

## MVP — "Never lose a lead, never lie"

Target: **6–8 weeks** with the client's availability data accessible.

### Must build
| # | Capability | Notes |
|---|---|---|
| 1 | WhatsApp Cloud API integration + webhook | Signature verification, dedupe, retry-safe |
| 2 | Conversation state per phone number | Slots persist across sessions |
| 3 | Intent routing | The 18 intents in `agent_design.md`; accident short-circuit |
| 4 | **Languages: English, Arabic, Russian** | Detection, switching, approved policy translations |
| 5 | FAQ from an approved knowledge base | Documents, age, mileage, Salik, delivery, hours, deposit |
| 6 | **Live availability via `check_availability()`** | The non-negotiable core |
| 7 | **Backend-generated itemised quote with expiry** | The single biggest trust differentiator |
| 8 | Vehicle recommendations with real prices | Max 3 options, list message |
| 9 | Document collection via WhatsApp media | Immediate move to encrypted storage; **human review queue** |
| 10 | Payment links + webhook-confirmed payment | Never infer success |
| 11 | **Atomic booking creation** | DB exclusion constraint; fails closed |
| 12 | Confirmation artefact (template + PDF) | Booking ID, terms recap, delivery window |
| 13 | Delivery scheduling + location pin capture | ETA and driver details template |
| 14 | **Extensions** (availability → price diff → payment → confirm) | With the substitution branch |
| 15 | **Accident + breakdown protocols** | Safety script, immediate escalation |
| 16 | **Human handoff with full context payload** | Customer never repeats themselves |
| 17 | Agent dashboard: inbox, take-over, escalation queue, document review, booking view | |
| 18 | Pricing/policy editable without a deploy | |
| 19 | Voice-note transcription + image classification | Very common in this market |
| 20 | Audit log of every tool call and money action | |

### Explicitly out of MVP
Autonomous refunds · autonomous document approval · autonomous cancellation · corporate/long-term
lease closing · damage adjudication · dynamic pricing · Telegram · Instagram DM · loyalty · crypto ·
customer app · outbound marketing campaigns.

### MVP success metrics
| Metric | Target |
|---|---|
| First response time | **< 10 seconds**, 24/7 *(market best: 5 minutes)* |
| Containment (resolved without a human) | ≥ 60% of conversations |
| Quote accuracy (quoted total = invoiced total) | **100%** — any deviation is a P1 bug |
| Double bookings | **0** |
| Lead → booking conversion vs. pre-launch baseline | +30% |
| Escalation SLA (CRITICAL) | Human in-thread < 2 minutes |

---

## V1 — "Own the whole rental, not just the sale"

Target: **+6–8 weeks after MVP.**

| # | Capability | Why now |
|---|---|---|
| 1 | **Proactive lifecycle messaging** — confirmation → delivery ETA → mid-rental check-in → return reminder (24h/2h) | Nobody in the market does this; cheap utility templates |
| 2 | **Salik + fines ingestion, auto-attribution and notification** | Turns the nastiest surprise into a routine update |
| 3 | **Deposit-refund tracker** with the 30-day countdown surfaced to the customer | Directly attacks the #1 complaint category |
| 4 | **Late-return proactive outreach** at grace-period start, including the insurance-void warning | Safety + revenue |
| 5 | Vehicle upgrades with price-difference flow (approval above a threshold) | Margin |
| 6 | Additional-driver flow (documents → human verify → contract amendment) | Common request |
| 7 | **Hindi/Urdu** | 2.6m Indian visitors + largest resident group |
| 8 | Automated document pre-checks (OCR: expiry, name match, licence class) — still human-approved | Cuts review time, keeps the human decision |
| 9 | Returning-customer recognition and one-tap rebooking | LTV |
| 10 | **AI-channel booking incentive** (SelfDrive's playbook — small discount for booking via the agent) | Proven adoption driver, measurable ROI |
| 11 | Agent analytics dashboard: containment, response time, conversion by intent, escalation reasons | Needed to tune |
| 12 | Handover/return photo sets attached to the booking | Ends damage disputes |

---

## V2 — "Automate what we've proved safe"

Target: **+3 months after V1**, gated on V1 metrics.

| # | Capability | Gate |
|---|---|---|
| 1 | **Automated document approval** for low-risk cases | Only after 3 months of ≥98% agreement between OCR checks and human reviewers |
| 2 | Self-service cancellation within policy | Only after cancellation rules are fully encoded and tested |
| 3 | Corporate / long-term lead qualification to a structured brief | Human still closes |
| 4 | Telegram as a second channel adapter | CIS market; same orchestration layer |
| 5 | Instagram DM adapter | ~41% of premium traffic originates there |
| 6 | Payment retry and dunning for monthly subscriptions | Long-term revenue protection |
| 7 | Upsell engine (insurance upgrade, extra mileage, chauffeur, child seat) driven by backend margin rules | Margin |
| 8 | Driver mobile flow for handover photos and status | Ops quality |
| 9 | Waitlist: notify when a requested unavailable car frees up | Recovers lost leads |
| 10 | Multi-brand / multi-branch support | Scale |

---

## Future — "Only if the data earns it"

- **Demand-based pricing suggestions** (suggested to a human, never auto-applied).
- **Fleet utilisation and demand forecasting** feeding purchasing decisions.
- **Telematics integration** — real odometer, fuel level, geofence alerts, automatic mileage overage.
- **RTA TARS contract integration** — digital rental contracts filed automatically.
- **Predictive maintenance scheduling** feeding the availability engine.
- **Voice AI** for inbound calls — only if call volume justifies it; there is no market evidence
  customers want it today.
- **White-label**: the same agent sold to other Dubai operators (the 3,494-company market is the
  actual TAM here — worth flagging to the boss as a second business model).

---

## Sequencing rationale

1. **Availability and pricing integration comes first.** Without it, everything else is a chatbot
   that lies. If the client's data lives in spreadsheets, week 1–2 is building the availability
   service, not the AI.
2. **Trust features before automation features.** A binding quote and zero double-bookings buy the
   credibility that lets us automate more later.
3. **After-booking (V1) before more automation (V2).** It is unserved by every competitor, cheap to
   build on utility templates, and it is where retention lives.
4. **Every autonomy increase is gated on measured accuracy**, not on a calendar date.
