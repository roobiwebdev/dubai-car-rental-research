# Technical Architecture

Deliberately short. One default stack, one reason per choice.

---

## 1. Flow

```
CUSTOMER (WhatsApp)
   │
   ▼
Meta WhatsApp Cloud API ──► WEBHOOK (verify signature, dedupe, enqueue)
   │
   ▼
CONVERSATION SERVICE ── loads state (slots, language, active booking, consent)
   │
   ▼
AI ORCHESTRATION (LLM + tool calling + guardrails)
   │
   ├─► KNOWLEDGE BASE (approved policies, per language)
   │
   ├─► BUSINESS TOOLS ──────────────────────────────────┐
   │      check_availability · get_quote · create_hold  │
   │      create_booking · modify_booking · extend      │
   │      lookup_bookings · store_documents             │
   │      create_payment_link · get_charges             │
   │      schedule_delivery · dispatch_roadside         │
   │      escalate                                      │
   │                                                    ▼
   │                                    RENTAL BACKEND (Postgres)
   │                                    fleet · availability · pricing rules
   │                                    bookings · customers · payments
   │                                                    │
   ├─► PAYMENT PROVIDER (links, pre-auth, refunds) ◄────┤
   ├─► DOCUMENT STORE (encrypted object storage)        │
   ├─► NOTIFICATIONS (WhatsApp templates, email)        │
   └─► HUMAN HANDOFF ──► AGENT DASHBOARD ───────────────┘
                              │
                              ▼
                   CONFIRMATION + LIFECYCLE FOLLOW-UP
```

**The load-bearing rule:** the LLM has **no direct database access**. Every fact enters the
conversation through a typed tool whose output the model may quote but not modify. Availability and
money are tool outputs, never model outputs.

---

## 2. Recommended stack (one default)

| Layer | Choice | One-line reason |
|---|---|---|
| **Messaging** | **Meta WhatsApp Cloud API** (direct) | Advanced interactive messages and Flows ship to Cloud first; no BSP margin or lock-in |
| BSP fallback | 360dialog or Twilio | Only if the client needs a partner-managed WABA or existing number migration |
| **AI model** | **Claude Sonnet 5** (`claude-sonnet-5`) for the conversation loop | Strong multilingual + tool use at chat latency and cost; escalate to `claude-opus-5` only for complex dispute summarisation |
| **Orchestration** | **TypeScript + Node** service with the Anthropic SDK's tool loop | One language across API and dashboard; typed tool schemas are the guardrail |
| **Backend/API** | **Next.js (App Router) on Vercel** — API routes + admin UI in one project | The team already works in this stack; edge-fast webhooks, zero ops |
| **Database** | **Postgres (Neon)** | Relational integrity is the product — booking conflicts must be a DB constraint; Neon branches make schema work safe |
| **Queue / async** | **Vercel Workflow (WDK)** or a Postgres-backed job table | Message processing, media download and template sends must survive retries |
| **Document storage** | **Vercel Blob** or **S3**, private + server-side encryption, signed short-lived URLs | Meta media URLs expire fast; documents must leave WhatsApp immediately |
| **Payments** | **Telr** or **Network International** (primary), Stripe if the client is already on it | UAE-native acquiring, AED settlement, hosted payment links, pre-auth support; Telr has Tabby BNPL pre-integrated |
| **CRM** | The Postgres `customers` table + the admin dashboard **is** the CRM for MVP | A second system is a sync problem; add HubSpot/Zoho only if sales demands it |
| **Rental backend** | **Integrate the client's existing system first.** If none: **HQ Rental** or a UAE-specific option (**Appic Fleet**, **Caryaati**) for Salik/fines/Mulkiya automation | Never rebuild fleet management; the agent needs an availability source, not a new ERP |
| **Notifications** | WhatsApp utility templates + Resend for email | Templates are the only way to message outside the 24h window |
| **Observability** | Structured logs + full conversation traces with tool inputs/outputs | Every quote and booking must be auditable after the fact |

### Explicitly not recommended
- **n8n / Make / Zapier as the core orchestrator** — fine for a prototype, wrong for money-handling
  logic that needs transactions, retries and audit trails.
- **A vector database in MVP** — the policy KB is small and must be *exactly* controlled. Use curated,
  versioned chunks, not similarity search over marketing copy.
- **Fine-tuning** — the constraints belong in tools and prompts, not in weights.
- **Building a rental ERP** — out of scope, and three UAE vendors already do it.

---

## 3. Tool contracts (the API the AI is allowed to touch)

```ts
check_availability(category?, vehicle_id?, start: ISO, end: ISO, area?)
  -> { units: [{vehicle_id, make, model, year, plate?, class}], alternatives: [...],
       checked_at: ISO }

get_quote(vehicle_id|category, start, end, delivery_location, extras[], promo?)
  -> { quote_id, line_items[], subtotal, vat, total, deposit{type,amount},
       mileage{cap_per_day, overage_rate}, not_included[], exclusions[], expires_at }

create_hold(quote_id)            -> { hold_id, expires_at }
create_booking(quote_id, customer_id, hold_id, payment_ref)
                                 -> { booking_id } | { error: CONFLICT, alternatives[] }
lookup_bookings(phone)           -> [ booking summaries ]
extend_booking(booking_id, new_end)         -> quote | CONFLICT + substitutes
modify_booking(booking_id, changes)         -> quote | requires_approval
store_documents(booking_id, media_ids[], type) -> { status: pending_review }
create_payment_link(quote_id, purpose)      -> { url, expires_at }
get_charges(booking_id)          -> { salik[], fines[], mileage_overage, deposit_status }
schedule_delivery(booking_id, location, window) -> { driver, eta }
dispatch_roadside(booking_id, location, symptom) -> { ticket_id, eta }
escalate(reason, priority, payload)         -> { ticket_id }
```

Every write tool is **idempotent** (client-supplied idempotency key) and **fails closed**.
`create_booking` runs in a serialisable transaction against a `(vehicle_id, tstzrange)` exclusion
constraint that includes turnaround buffers.

---

## 4. Data model

| Entity | Purpose | Key fields |
|---|---|---|
| **Customer** | One per person, keyed on phone | id, phone, name, language, residency_status, nationality, licence_country, dob, vip, consent_at, created_at |
| **Document** | A collected identity/licence document | id, customer_id, booking_id, type (passport/visa/emirates_id/licence/idp/card), storage_key, expiry_date, status (pending/verified/rejected), reviewed_by, reviewed_at |
| **VehicleCategory** | Class-level rules and pricing | id, name, min_age, min_licence_years, deposit_default, mileage_cap_per_day, overage_rate |
| **Vehicle** | A physical unit | id, category_id, make, model, year, plate, vin, mulkiya_expiry, status (available/rented/maintenance/damaged), current_odometer, home_location |
| **Availability** | Derived, not stored raw | computed from bookings + buffers + Maintenance + Damage blocks |
| **Booking** | The contract | id, customer_id, vehicle_id, category_id, start_at, end_at, actual_return_at, delivery_location, dropoff_location, status (quoted/held/confirmed/active/returned/cancelled), quote_id, rta_contract_ref |
| **Quote** | An immutable priced offer | id, booking_draft, line_items jsonb, total, deposit, expires_at, created_at |
| **PricingRule** | How money is calculated | id, category_id, rate_type (daily/weekly/monthly), base_rate, season_range, min_days, discount_pct, delivery_zone_fees jsonb, valid_from/to |
| **Payment** | Money in | id, booking_id, provider, provider_ref, amount, currency, type (rental/extension/deposit/fine/salik), status, paid_at |
| **Deposit** | The hold and its release | id, booking_id, type (cash/pre_auth/none), amount, held_at, release_due_at, released_at, deductions jsonb |
| **Delivery** | A movement of a vehicle | id, booking_id, type (delivery/collection), address_text, geo_point, window_start/end, driver_id, status, handover_photos[] |
| **Location** | Delivery zones and fees | id, name, zone, is_airport, fee, eta_minutes |
| **Maintenance** | Planned downtime | id, vehicle_id, type (service/cleaning/mulkiya/repair), start_at, end_at |
| **Damage** | Incident record | id, vehicle_id, booking_id, reported_at, description, photos[], police_report_ref, cost, liability (customer/insurer/company), status |
| **Fine** | A traffic violation | id, vehicle_id, booking_id, authority_ref, occurred_at, amount, admin_fee, status (unbilled/billed/paid/disputed) |
| **SalikCharge** | Toll pass-through | id, vehicle_id, booking_id, crossed_at, gate, amount, admin_fee |
| **Conversation** | One per customer thread | id, customer_id, channel, language, state jsonb (slots), status (ai/human/closed), assigned_agent_id |
| **Message** | Every inbound/outbound message | id, conversation_id, direction, type, body, media_key, template_name, tool_calls jsonb, created_at |
| **SupportTicket** | An escalation | id, conversation_id, booking_id, reason, priority, payload jsonb, assigned_to, sla_due_at, resolved_at |

**Constraints that matter:**
- Exclusion constraint on `Booking(vehicle_id, tstzrange(start_at - buffer, end_at + buffer))` for
  statuses `held/confirmed/active` → **double-booking is structurally impossible**.
- `Quote.expires_at` enforced server-side; an expired quote cannot become a booking.
- `Document.storage_key` never appears in `Message.body`.

---

## 5. Admin / operations requirements

### MVP (required at launch)
| Area | Requirement |
|---|---|
| **Conversations** | Live inbox of all WhatsApp threads; AI/human status; **one-click take-over** and release |
| **Escalations** | Queue with priority and SLA timers; full handoff payload visible |
| **Bookings** | List/detail; create, modify, cancel, extend; status transitions; audit trail |
| **Fleet** | Vehicle list with real-time status; block a vehicle (maintenance/damage) instantly |
| **Availability** | Calendar view per vehicle and per category, showing buffers |
| **Pricing** | Edit rates, seasons, minimum days, delivery-zone fees, deposit amounts — **without a deploy** |
| **Documents** | Review queue: approve/reject with a reason, expiry capture |
| **Payments** | Payment status per booking, resend link, mark cash received; **refunds require a named approver** |
| **Deliveries** | Dispatch board: assign driver, set window, upload handover photos |
| **Customers** | Profile, history, language, VIP flag, blocklist |
| **Knowledge base** | Edit approved policy answers per language, with versioning |
| **Audit log** | Who/what/when for every money and booking action, plus every AI tool call |

### V1
Agent analytics (containment rate, response time, conversion by intent), Salik/fines ingestion and
auto-attribution, deposit-refund tracker with the 30-day countdown, template management, canned
responses, shift/on-call routing, revenue and utilisation dashboards.

### V2+
Demand-based pricing suggestions, fleet utilisation forecasting, driver mobile app, telematics
integration, RTA **TARS** contract integration, multi-branch and multi-brand support.

---

## 6. Security and privacy

The data at stake is passports, Emirates IDs, visas, driving licences and payment references —
regulated under **UAE Federal Decree-Law 45/2021 (PDPL)**, effective January 2022.

### Risks and controls

| Risk | Control |
|---|---|
| Identity documents sitting in WhatsApp threads on staff phones | Download media on receipt, store **only** in an encrypted private bucket, keep a pointer in the message record. Staff never handle raw media |
| Over-collection | Collect only the documents required for that customer type. Nothing "just in case" |
| Indefinite retention | Retention policy per document type; automatic deletion after rental + statutory period. PDPL requires data not be kept longer than necessary |
| Consent | Explicit, purpose-specific consent captured in-conversation on first contact, stored with a timestamp, revocable |
| Payment data | **Never** touches our systems. Hosted payment links only; store provider references, never PANs |
| AI over-access | The model sees only the fields a tool returns. No raw DB access, no cross-customer queries. Document *contents* are never placed in the model context — only status |
| Staff over-access | Role-based access: agents see their conversations; finance sees payments; only named approvers can refund |
| Prompt injection via customer messages | Message content is data, never instruction. Tool authorisation is enforced server-side, not by the prompt |
| Data leakage in responses | Output filter blocking document numbers, card numbers, internal IDs and tool errors |
| Audit | Immutable log of every tool call with inputs, outputs and the resulting message; every money action attributed to a person or the agent |
| Breach | Breach-notification process to the UAE Data Office, per PDPL |
| Cross-border transfer | Document the AI provider's processing location and include it in the privacy notice |

**Not in scope here:** general corporate security (endpoint, network, SSO). That is a separate
workstream; this section covers what *this product* introduces.
