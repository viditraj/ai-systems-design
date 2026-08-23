# Detailed Architecture — Expedia AI Travel Agent

## 1. Architecture Goal

Design a production-grade conversational travel agent inspired by Expedia Group's AI travel experiences. The agent converts natural-language travel goals into reliable search, recommendation, itinerary, booking, payment, and post-booking workflows.

The central principle is:

> **The LLM can reason and propose actions, but deterministic services own truth, authorization, pricing, payment, and booking state.**

The architecture is provider-agnostic at the core and uses adapters around real external travel ecosystems such as flight distribution, hotel inventory, maps/places, payments, and messaging providers.

---

## 2. Example User Flow

```text
User:
"Plan 5 days in Paris from Mumbai under ₹2 lakh.
 Prefer direct flights and a 4-star hotel near the city center.
 Show me options and book after I approve."

        ↓
Intent + Constraint Extraction
        ↓
Search Flights + Hotels + Activities
        ↓
Candidate Normalization
        ↓
Ranking / Personalization
        ↓
Itinerary Generation
        ↓
Live Price + Availability Revalidation
        ↓
User Approval
        ↓
Booking Workflow
        ↓
Payment
        ↓
Booking Confirmation
        ↓
Trip Management
```

---

## 3. High-Level Architecture

```text
                                   USER
                                     │
                                     ▼
                           ┌──────────────────┐
                           │  Web / Mobile    │
                           │  / Voice / API   │
                           └────────┬─────────┘
                                    │
                                    ▼
                           ┌──────────────────┐
                           │   API Gateway    │
                           │ Auth / WAF /     │
                           │ Rate Limit       │
                           └────────┬─────────┘
                                    │
                                    ▼
                    ┌─────────────────────────────┐
                    │     Travel Agent            │
                    │  LangGraph / Workflow       │
                    │  Durable State              │
                    └──────────────┬──────────────┘
                                   │
            ┌──────────────────────┼────────────────────────┐
            │                      │                        │
            ▼                      ▼                        ▼
     ┌──────────────┐      ┌──────────────┐        ┌──────────────┐
     │ Intent /     │      │ Travel       │        │ User Memory  │
     │ Constraint   │      │ Planner      │        │ & Profile    │
     │ Extraction   │      │              │        │              │
     └──────┬───────┘      └──────┬───────┘        └──────────────┘
            │                     │
            └──────────┬──────────┘
                       ▼
               ┌───────────────┐
               │ Search Router │
               └───────┬───────┘
                       │
        ┌──────────────┼──────────────────┐
        ▼              ▼                  ▼
    Flights          Hotels           Activities
        │              │                  │
        ▼              ▼                  ▼
  Provider Adapters / Supplier APIs / Inventory
        │              │                  │
        └──────────────┼──────────────────┘
                       ▼
                Normalization Layer
                       │
                       ▼
                Ranking / Filtering
                       │
                       ▼
                Itinerary Composer
                       │
                       ▼
              Price + Availability
                 Revalidation
                       │
                       ▼
                 Policy Engine
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
          Auto-Allow          Approval
             │                   │
             │             Human Confirmation
             └─────────┬─────────┘
                       ▼
             Booking Orchestrator
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Payment      Flight        Hotel
       Service      Booking       Booking
          │            │            │
          └────────────┼────────────┘
                       ▼
                Booking State Store
                       │
                       ▼
              Notifications / Trip UI
```

---

## 4. Requirements

### Functional

- Understand natural-language travel requests.
- Extract dates, locations, travelers, budget, preferences, and constraints.
- Search flights, hotels, activities, and transportation.
- Compare and rank options.
- Generate multi-day itineraries.
- Explain recommendations.
- Revalidate live price and availability before booking.
- Obtain user confirmation for booking.
- Execute bookings and payments.
- Support cancellation and modification.
- Track trip status.
- Handle provider failures and disruptions.

### Non-functional

Assume:

```text
10M+ monthly active travelers
100K+ peak concurrent sessions
Several thousand search requests/sec
Hundreds of booking transactions/sec
Multi-region deployment
99.99% availability for booking APIs
P95 search latency < 3 seconds where providers permit
P95 conversational response < 5 seconds for non-streaming generation
```

Booking correctness is more important than conversational latency.

---

## 5. Agent Workflow

Do not use an unrestricted autonomous loop. Use a durable state machine.

```text
REQUESTED
    │
    ▼
UNDERSTANDING
    │
    ▼
SEARCHING
    │
    ▼
RANKING
    │
    ▼
ITINERARY_READY
    │
    ├────────── user changes constraints ──────┐
    │                                          │
    ▼                                          │
REVALIDATING                                  │
    │                                          │
    ▼                                          │
AWAITING_APPROVAL ◄────────────────────────────┘
    │
    ├── rejected → CANCELLED
    │
    ▼
BOOKING
    │
    ▼
PAYMENT
    │
    ▼
CONFIRMING
    │
    ▼
BOOKED
    │
    ├── change request → MODIFYING
    ├── cancellation → CANCELLING
    └── disruption → DISRUPTION_HANDLING
```

Persist every transition so workers can crash and resume safely.

---

## 6. Intent and Constraint Extraction

Convert natural language into structured constraints before calling providers.

```json
{
  "origin": "BOM",
  "destination": "CDG",
  "departure_date": "2026-10-12",
  "return_date": "2026-10-17",
  "travelers": 2,
  "budget": {
    "currency": "INR",
    "max": 200000
  },
  "flight_preferences": {
    "direct": true
  },
  "hotel_preferences": {
    "stars": 4,
    "area": "city_center"
  }
}
```

The LLM proposes this structure; deterministic validation checks dates, currencies, traveler counts, required fields, and supported constraints.

---

## 7. Search Architecture

Use parallel search rather than serial provider calls.

```text
                 Search Request
                       │
                       ▼
                 Search Router
                       │
       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
   Flight Search    Hotel Search    Activity Search
       │               │                │
 ┌─────┴─────┐    ┌────┴─────┐      Providers
 ▼           ▼    ▼          ▼
Provider A Provider B Provider A Provider B
       │               │                │
       └───────────────┼────────────────┘
                       ▼
                Result Normalizer
                       ▼
                 Deduplication
                       ▼
             Hard Constraint Filter
                       ▼
                Ranking Service
```

Provider-specific APIs must be hidden behind adapters. This allows the platform to replace a supplier, use multiple suppliers, or fail over without changing the agent.

---

## 8. Real Travel Provider Integration

The architecture should model real-world provider categories rather than fictional internal APIs.

Examples include:

- **Amadeus** — flight and hotel travel APIs.
- **Duffel** — airline/NDC distribution and booking APIs.
- **Booking.com** — accommodation inventory ecosystem.
- **Google Maps Platform** — places, geocoding and routing.
- **Stripe** — payment processing where applicable.
- **Twilio / SendGrid** — traveler notifications.

These are external integrations, not claims about Expedia's proprietary internal implementation.

```text
                 Expedia Agent
                      │
                Provider Gateway
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
   Flight Adapter  Hotel Adapter  Places Adapter
       │              │              │
   Amadeus /       Booking.com /    Google Maps
   Duffel          other suppliers  Platform
```

The gateway handles authentication, request normalization, timeout policies, retries, circuit breakers, caching, rate limits, and provider-specific error mapping.

---

## 9. Candidate Ranking

Search returns many candidates. Ranking should be a separate deterministic/ML service rather than asking the LLM to rank thousands of options.

Example ranking features:

```text
score =
  price_weight
+ duration_weight
+ preference_match
+ schedule_quality
+ hotel_location_score
+ cancellation_flexibility
+ supplier_quality
+ historical_user_preference
```

A learned ranking model can replace the simple scoring function as the system matures.

The LLM should explain the selected candidates, not become the source of numerical truth.

---

## 10. Personalization

Maintain a structured traveler profile:

```json
{
  "preferred_cabin": "economy",
  "preferred_airlines": ["..."],
  "hotel_star_preference": 4,
  "preferred_areas": ["city_center"],
  "dietary_preferences": ["vegetarian"],
  "seat_preference": "aisle"
}
```

Separate explicit preferences from inferred preferences.

```text
Explicit:
"I prefer aisle seats."

Inferred:
User repeatedly selects aisle seats.
```

Inferred preferences should have confidence scores and should never override an explicit current request.

---

## 11. Itinerary Planning

The itinerary planner combines booked/selected travel segments with activities and geographic constraints.

```text
Flights
   │
Hotels ──────┐
   │          │
Activities   │
   │          ▼
   └──────► Itinerary Optimizer
                 │
                 ├── travel time
                 ├── opening hours
                 ├── user preferences
                 ├── budget
                 ├── geographic proximity
                 └── schedule constraints
                 │
                 ▼
              Day Plan
```

Use routing/places APIs for factual travel time and location data. Do not let the LLM invent distances or opening hours.

---

## 12. Price and Availability Revalidation

This is one of the most important travel-system concerns.

```text
Search Result
     │
     ▼
User selects itinerary
     │
     ▼
Fetch live offer / fare
     │
     ├── price unchanged → continue
     │
     ├── price changed → show new price
     │
     └── unavailable → search alternatives
```

Never book directly from stale search results.

Store an offer expiry timestamp where provided by the supplier.

---

## 13. Booking Orchestrator

Booking is a distributed transaction across external providers.

```text
                  Booking Request
                         │
                         ▼
                  Validate State
                         │
                         ▼
                 Revalidate Offers
                         │
                         ▼
                    Payment Auth
                         │
                         ▼
                 Flight Booking
                         │
                         ▼
                  Hotel Booking
                         │
                         ▼
                Activity Booking
                         │
                         ▼
                  Confirmation
```

Do not assume a global ACID transaction exists across suppliers.

Use a saga-like workflow with compensating actions.

Example:

```text
Flight BOOKED
Hotel FAILED
     │
     ▼
Attempt flight cancellation
     │
     ▼
Refund / compensation workflow
     │
     ▼
Find alternative hotel
```

Every step must have an explicit success, failure, retry, and compensation policy.

---

## 14. Idempotency

Booking APIs must be protected against duplicate execution.

```text
idempotency_key =
  user_id + trip_id + booking_attempt_id + component
```

```text
Booking Worker
      │
      ▼
Idempotency Store
   │          │
exists       missing
   │          │
return       execute
result          │
                ▼
           store result
```

This protects against worker crashes, network retries, duplicate messages, and user double-clicks.

---

## 15. Payment Architecture

Payment information should remain outside the LLM context.

```text
Agent
  │
  ▼
Booking Service
  │
  ▼
Payment Service
  │
  ▼
Payment Provider
```

The LLM should see only safe transaction state such as:

```text
PAYMENT_AUTHORIZED
PAYMENT_FAILED
PAYMENT_REQUIRES_ACTION
```

Never place raw card numbers, CVVs, or payment secrets into prompts, logs, memory, or vector databases.

---

## 16. Human Approval

For purchases, the agent should present a precise booking summary.

```text
Trip: Mumbai → Paris
Dates: Oct 12–17
Travelers: 2
Flight: ₹72,400
Hotel: ₹81,200
Activities: ₹18,500
Taxes/fees: ₹9,800
Total: ₹181,900

[Approve booking]
[Change options]
[Cancel]
```

The approval is attached to a specific immutable booking proposal/version. A changed price or itinerary requires fresh approval.

---

## 17. RAG for Travel Knowledge

RAG is useful for policies and explanations, but not for live inventory.

Use RAG for:

- baggage policies
- cancellation rules
- loyalty-program documentation
- travel policies
- destination guidance
- visa/entry guidance where authoritative sources are available
- supplier policy documents

Use APIs for:

- live prices
- availability
- booking status
- flight status
- hotel inventory
- payment status

```text
Policy Question → RAG
Live Transaction → Authoritative API
```

---

## 18. Memory

Separate memory types.

```text
Conversation State → Durable Workflow Store
Traveler Preferences → Profile DB
Past Trips → Transactional DB
Travel Knowledge → RAG Index
Current Booking → Booking DB / Supplier APIs
```

Do not use vector memory as the authoritative booking record.

---

## 19. Failure Handling

External travel providers are unreliable compared with internal services.

Use:

- timeouts
- retries only for safe/idempotent operations
- exponential backoff
- circuit breakers
- bulkheads
- provider health scoring
- fallback suppliers
- dead-letter queues
- reconciliation jobs

Example:

```text
Provider Timeout
      │
      ▼
Retry with backoff
      │
      ├── success → continue
      │
      └── failure
            │
            ▼
      Circuit Breaker
            │
            ▼
      Fallback Provider
```

---

## 20. Flight Disruption Handling

A production travel agent must continue working after booking.

```text
Airline Event / Flight Status
           │
           ▼
      Event Ingestion
           │
           ▼
      Trip Matcher
           │
           ▼
     Impact Analyzer
           │
     ┌─────┴─────┐
     ▼           ▼
No impact      Impact
                 │
                 ▼
          Alternative Search
                 │
                 ▼
          User Notification
                 │
                 ▼
          Rebooking Workflow
```

The agent can explain options, but deterministic services should calculate eligibility, fare differences, and cancellation/rebooking rules.

---

## 21. Security and Guardrails

Travel agents have valuable user data and financial side effects.

Controls include:

- OAuth/OIDC authentication
- tenant/user authorization
- PCI-aware payment isolation
- PII encryption
- secrets manager
- tool allow-list
- schema validation
- prompt-injection defenses
- supplier API credential isolation
- audit logging
- fraud/risk checks
- rate limiting
- anomaly detection

The LLM must never receive unrestricted booking or payment credentials.

---

## 22. Tool Calling

Tools should be narrowly scoped.

```text
search_flights
search_hotels
search_activities
get_offer
get_hotel_details
create_booking
cancel_booking
modify_booking
get_trip_status
get_flight_status
```

Avoid one generic tool such as:

```text
execute_travel_api(request)
```

Narrow tools make authorization, validation, observability, and testing easier.

---

## 23. Model Gateway

Use a model gateway instead of coupling the agent to one LLM.

```text
                     Agent
                       │
                       ▼
                 Model Gateway
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Small LLM     Main LLM    Reasoning LLM
          │            │            │
       routing      dialogue      planning
       extraction    synthesis     complex tasks
```

Track:

- model latency
- input/output tokens
- cost
- tool-call accuracy
- structured-output failures
- fallback rate
- quality scores

---

## 24. Observability

Observe both conventional distributed-system health and AI quality.

### System metrics

- request rate
- P50/P95/P99 latency
- error rate
- provider timeout rate
- circuit-breaker trips
- queue depth
- worker utilization
- booking success rate
- payment success rate
- cancellation success rate

### AI metrics

- intent extraction accuracy
- constraint extraction accuracy
- tool-selection accuracy
- tool-call failure rate
- hallucination rate
- recommendation acceptance rate
- itinerary edit rate
- user satisfaction
- groundedness
- prompt-injection detection rate
- LLM token usage
- cost/request

### Business metrics

- search-to-book conversion
- itinerary-to-book conversion
- booking abandonment
- average booking value
- repeat travelers
- cancellation rate
- disruption recovery rate

Use distributed tracing to connect:

```text
user request
 → agent run
 → LLM calls
 → tool calls
 → supplier calls
 → payment
 → booking
```

---

## 25. Evaluation

Create an offline evaluation set containing realistic travel requests.

Evaluate:

```text
Request Understanding
        ↓
Constraint Extraction
        ↓
Search Selection
        ↓
Ranking
        ↓
Itinerary Quality
        ↓
Tool Selection
        ↓
Policy Compliance
        ↓
Final Response
```

Useful metrics include:

- constraint accuracy
- retrieval precision/recall for policy RAG
- tool-call accuracy
- booking-plan validity
- groundedness
- task completion rate
- unsafe-action rate
- price/availability consistency
- latency
- cost

Run regression evaluations whenever prompts, models, ranking models, or tool schemas change.

---

## 26. Caching

Cache only data where staleness is acceptable.

Good candidates:

- destination metadata
- hotel static metadata
- place information
- policy documents
- common search metadata

Be careful with:

- live flight prices
- live hotel availability
- booking status

Use short TTLs and supplier-defined expiry where applicable.

Semantic caching can help repetitive informational questions but should not be used blindly for transactional booking requests.

---

## 27. Scalability

Stateless services should scale horizontally.

```text
                    Load Balancer
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
      Agent API       Agent API      Agent API
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                  Durable Workflow
                         │
                    Event / Queue
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       Search          Booking        Notification
       Workers         Workers         Workers
```

Partition workloads by traveler/session where useful and isolate booking workers from exploratory search workloads.

---

## 28. Multi-Region Architecture

For global travel traffic:

```text
             Global DNS / Anycast
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
       Region A                Region B
          │                       │
     Agent Stack              Agent Stack
          │                       │
     Regional Cache          Regional Cache
          │                       │
          └──────────┬────────────┘
                     ▼
              Global Data Layer
```

Keep booking workflows region-resilient and use durable state replication. Avoid active-active writes to a transactional record unless the consistency model is explicitly designed.

---

## 29. Key Trade-offs

### LLM vs deterministic planner

Use the LLM for natural language and flexible reasoning. Use deterministic workflow logic for transactions.

### Single supplier vs multiple suppliers

Single supplier simplifies integration. Multiple suppliers improve inventory coverage and resilience but add normalization and reconciliation complexity.

### Search freshness vs cache efficiency

Long TTL improves cost and latency but increases stale-price risk. Transactional data should always be revalidated.

### Autonomous booking vs human confirmation

Autonomous booking improves convenience but increases financial and safety risk. Require explicit confirmation for purchases unless the user has configured trusted autonomous-booking policies.

### RAG vs APIs

RAG is excellent for policy and knowledge. APIs are authoritative for live transactional state.

---

## 30. Final Architecture Principle

```text
             LLM
              │
      Understand / Plan / Explain
              │
              ▼
       Durable Workflow
              │
       ┌──────┴──────┐
       ▼             ▼
   Policy Engine   Tool Gateway
       │             │
       └──────┬──────┘
              ▼
      Deterministic Services
              │
     ┌────────┼─────────┐
     ▼        ▼         ▼
  Search    Payment   Booking
     │        │         │
     └────────┼─────────┘
              ▼
       Authoritative State
```

The system should remain safe and correct even when the LLM makes a poor decision. The model is an intelligent interface and planner, not the system of record and not the final authority for irreversible travel transactions.
