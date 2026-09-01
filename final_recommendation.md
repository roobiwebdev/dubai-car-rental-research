# WHAT WE SHOULD BUILD

The executive technical recommendation. Everything here traces back to evidence in the other files.

---

## The one-paragraph version

Dubai has **3,494 registered rental companies** and **71,040 rental vehicles** competing for
**19.59m visitors**, and the industry's default sales channel is a **human typing on WhatsApp**. The
best published response SLA we found in the whole market is **5 minutes** (Octane.Rent). The region's
only real rental AI — **SelfDrive's SIA**, 40+ languages, booking-capable — runs on **web and app,
not WhatsApp**. So the opportunity is not "add AI to a crowded AI market"; it is **be the first
correct AI on the channel where 90% of UAE customers already are.** The way to win is not
conversational polish — it is *accuracy*: a backend-verified availability answer and a **binding,
itemised quote** delivered in under 10 seconds, because the market's loudest documented failure is a
price agreed on WhatsApp that does not match the invoice.

---

## 1. What the agent SHOULD do

1. **Answer instantly, 24/7, in the customer's language** (EN/AR/RU at MVP).
2. **Qualify a rental in a few messages** — vehicle/class, dates, delivery location, residency, age.
3. **Check real availability** through a backend tool, every time, and offer verified alternatives
   when the answer is no.
4. **Issue a binding, itemised, timestamped quote** with a stated expiry — rental, delivery, VAT,
   deposit type and amount, mileage cap and overage rate, and what is *not* included (Salik, fines,
   fuel, excluded damage).
5. **Collect documents** in-chat and move them immediately into encrypted storage.
6. **Send a payment link** and confirm payment only on a provider webhook.
7. **Create the booking atomically** and issue a real confirmation artefact with a booking ID.
8. **Coordinate delivery** — capture a WhatsApp location pin, confirm a window, send driver details.
9. **Support the whole rental**: extensions, upgrades, Salik and fine notifications, return
   reminders, deposit-refund status.
10. **Run the accident and breakdown safety protocols** and escalate instantly.
11. **Hand off to a human with complete context**, so the customer never repeats themselves.

## 2. What the agent should NOT do

1. Never state availability without a tool call in the same turn.
2. Never state a price, fee, discount, deposit or total that the backend did not produce — and never
   do arithmetic on money.
3. Never say "confirmed" before `create_booking()` succeeds.
4. Never say "payment received" without a payment-provider event.
5. Never approve or reject an identity document autonomously (MVP).
6. Never issue a refund, release a deposit, or waive a charge.
7. Never adjudicate fault, damage liability or insurance coverage — including "is a flat tyre
   covered?" (tyres and rims are commonly excluded from both standard cover *and* CDW).
8. Never state a policy that is not in the approved knowledge base.
9. Never echo passport, Emirates ID, licence or card numbers back into the chat.
10. Never reveal internal systems, tool names, errors or other customers' data.
11. Never commit a corporate or 3+ month lease autonomously.
12. Never argue with an angry customer — de-escalate once, then hand off.

## 3. Core customer journey

```
INQUIRY (free text, any language, often partial)
   → SLOT FILL (vehicle/class · dates · location · residency · age)
   → AVAILABILITY  [backend]
   → ITEMISED QUOTE + EXPIRY  [backend]
   → TERMS RECAP (deposit · mileage · exclusions · not-included)
   → DOCUMENTS (in-chat capture → encrypted store → human review)
   → PAYMENT LINK → webhook-confirmed
   → BOOKING CREATED  [atomic]
   → CONFIRMATION ARTEFACT + DELIVERY WINDOW
   → DELIVERY (location pin · driver ETA · handover photos)
   → IN-RENTAL (extensions · upgrades · Salik · fines · roadside · accident)
   → RETURN (reminders · inspection photos)
   → DEPOSIT REFUND STATUS (30-day rule, proactively communicated)
   → REBOOK
```

## 4. Core tools/functions

`check_availability` · `get_quote` · `create_hold` · `create_booking` · `lookup_bookings` ·
`modify_booking` · `extend_booking` · `store_documents` · `create_payment_link` · `get_charges` ·
`schedule_delivery` · `dispatch_roadside` · `escalate`

All idempotent, all fail-closed, all audited. Full signatures in `technical_architecture.md §3`.

## 5. Required backend integrations

| Integration | Purpose | Priority |
|---|---|---|
| **Rental/fleet system (client's existing, or HQ Rental / Appic Fleet / Caryaati)** | Availability + pricing source of truth | **Blocking — week 1** |
| **Meta WhatsApp Cloud API** | The channel | Blocking |
| **Payment gateway (Telr or Network International)** | Payment links, pre-auth, refunds, AED settlement | Blocking |
| Encrypted object storage (Vercel Blob / S3) | Documents | Blocking |
| Salik / traffic-fine feed | After-booking charges | V1 |
| Driver dispatch (can be the dashboard in MVP) | Delivery | MVP-lite |
| RTA TARS | Regulated rental contracts | Future |

## 6. Required database entities

`Customer · Document · VehicleCategory · Vehicle · Booking · Quote · PricingRule · Payment ·
Deposit · Delivery · Location · Maintenance · Damage · Fine · SalikCharge · Conversation · Message ·
SupportTicket`

The one that matters most: an **exclusion constraint on `Booking(vehicle_id, time_range + buffers)`**
so double-booking is structurally impossible rather than prompt-discouraged.

## 7. Human handoff rules

**Immediate escalation:** accident/injury (CRITICAL) · breakdown · damage dispute · deposit or refund
dispute · payment dispute · legal/insurance matter · angry customer · explicit request for a person.
**Approval-gated:** cancellations · refunds · document rejection · upgrades above threshold ·
additional drivers · corporate and long-term leases.
**Confidence-gated:** anything the agent is unsure of.

The handoff payload carries customer, booking, financial state, documents status, location, full
transcript and a one-line agent summary. **SLA: human in-thread within 2 minutes for CRITICAL.**

## 8. MVP features

The 20 items in `mvp_scope.md`. In one line: **WhatsApp in → verified availability → binding quote →
documents → payment → atomic booking → delivery → extensions → accident/breakdown protocols → human
handoff with context → an admin dashboard that can take over any conversation.**
Languages **EN/AR/RU**. Timeline **6–8 weeks** once availability data is accessible.

## 9. Best competitor ideas to borrow

| Idea | From | Why |
|---|---|---|
| **AI-channel booking discount** (AED 50–100 off) | SelfDrive/SIA | Proven adoption driver; makes ROI measurable |
| **Pre-authorisation instead of a cash deposit** (AED 2,000 standard) | Superior Car Rental | Removes the #1 objection without carrying risk; automatable via payment link |
| **Fully published fee schedule** (mileage, per-km, delivery, deductible) | Rentico | The template for our itemised quote |
| **Extension-conflict substitution** ("that car is booked, but you may exchange for another available model") | Luxury Supercars Dubai | The most graceful handling of the hardest operational conflict |
| **7-day deposit refund** | Saadatrent | 3–4× faster than the market; a headline trust feature |
| **Zero deposit + unlimited mileage** on selected classes | NCK, eZhire | Kills the two biggest hidden-cost complaints |
| **Three-document digital KYC** (ID/passport + licence + card) | eZhire | The exact document set; no more, no less |
| **Delivery SLA as a promise** (30/60 minutes) | Panorama, Superior | Immediacy is the luxury segment's real currency |
| **Airport meet-and-greet with luggage help** | Star Empire | Converts the arrivals-hall moment |
| **Telegram as a second channel** | Renty, Octane | Direct line to 2.89m CIS visitors; cheap second adapter |
| **Explicit exclusion disclosure** (tyres/rims not covered) | Superior | Counter-intuitive, and exactly why it builds trust |
| **Pay-now 2% discount** | Quick Lease | Converts a chat quote into cash today |

## 10. Features we can improve beyond competitors

| # | Improvement | Beats |
|---|---|---|
| 1 | **Sub-10-second, backend-verified reply, 24/7** | 30× the market's best human SLA (Octane's 5 min) |
| 2 | **Binding itemised quote with a quote ID and expiry** | Fixes the documented "WhatsApp price ≠ invoice" failure — the market's #1 trust problem |
| 3 | **Structurally impossible double-booking** | Replaces manual spreadsheet/WhatsApp-group checking |
| 4 | **Proactive after-booking lifecycle** — delivery ETA, return reminder, Salik/fines, deposit countdown | **No competitor researched does any of this** |
| 5 | **Deposit-refund transparency against the legal 30-day rule** | The loudest complaint category, turned into a feature |
| 6 | **Genuine EN/AR/RU on WhatsApp** | SIA has 40+ languages but is not on WhatsApp; WhatsApp desks are English-first |
| 7 | **Context-preserving handoff** | Nobody has this; customers currently repeat everything |
| 8 | **Handover + return photo evidence on every booking** | Ends damage disputes before they start |
| 9 | **Secure document handling** (out of WhatsApp, encrypted, retention-limited) | PDPL-compliant vs. passports living in staff chat threads |
| 10 | **Marketplace lead reconciliation** (handling stale OneClickDrive price anchors honestly) | Nobody has a defined approach |

## 11. Recommended technical stack

**WhatsApp Cloud API** (direct) → **Next.js on Vercel** (webhook + API + admin) →
**Claude Sonnet 5** with typed tool-calling → **Postgres on Neon** (with the booking exclusion
constraint) → **Telr or Network International** for payment links and pre-auth →
**Vercel Blob / S3** encrypted document store → WhatsApp **utility templates** for lifecycle
messaging → structured tool-call audit logging. Integrate the client's **existing fleet system**;
only if none exists, adopt **HQ Rental** or a UAE-specific option (**Appic Fleet**, **Caryaati**) for
Salik/fines/Mulkiya automation.

One reason each in `technical_architecture.md §2`. Not recommended: n8n/Zapier as core orchestrator,
vector DB in MVP, fine-tuning, or building a rental ERP.

## 12. Recommended implementation order

| Phase | Weeks | Deliverable | Gate to proceed |
|---|---|---|---|
| **0. Discovery** | 1 | Client's current fleet/availability source, pricing rules, document policy, escalation staffing, WhatsApp number status | We know where availability truth lives |
| **1. Availability + pricing service** | 1–2 | `check_availability` and `get_quote` working against real client data, with the DB exclusion constraint | A quote is reproducible and correct 100% of the time |
| **2. WhatsApp pipe** | 1 | Cloud API, webhook, media handling, state store, audit log — echo-level, no AI | Messages never lost or duplicated |
| **3. Conversational core** | 2 | Router, slot filling, FAQ from the KB, EN/AR/RU, tool-calling loop with guardrails | Correct answers, zero invented facts in a 200-conversation test |
| **4. Booking + payment** | 1–2 | Documents, payment links, atomic booking, confirmation artefact | Zero double-bookings; quote = invoice, always |
| **5. Delivery + extensions + safety** | 1 | Delivery scheduling, extension flow with substitution, accident/breakdown protocols | Safety scripts verified by the operations team |
| **6. Dashboard + handoff** | 1–2 | Inbox, take-over, escalation queue with payload, document review, pricing editor | Ops team can run a day without a developer |
| **7. Pilot** | 2 | Live on a **subset of traffic** with a human watching every conversation | Containment ≥60%, quote accuracy 100%, zero double-bookings |
| **8. Full launch + V1** | ongoing | Proactive lifecycle messaging, Salik/fines, deposit tracker, Hindi/Urdu | Metrics in `mvp_scope.md` |

**Critical path warning for the boss:** Phase 1 is the whole project's risk. If the client's
availability lives in a spreadsheet or in a manager's head, the AI cannot be truthful and the project
should not proceed to Phase 3 until that is fixed. **Do not build the conversation before the source
of truth exists.**

---

## The pitch, in three sentences

Dubai's rental market runs on WhatsApp and nobody has put a working AI on it. We can answer in under
10 seconds instead of 5 minutes, guarantee the quote matches the invoice — fixing the industry's
single biggest complaint — and be the only operator that keeps talking to the customer after the
booking, through Salik, fines, extensions and the deposit refund. The technical requirement is
unglamorous but absolute: **the AI never invents availability or price; it reads them from a backend
that cannot double-book.**
