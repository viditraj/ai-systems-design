# Mock Interview — Expedia AI Travel Agent

## 1. Why use an LLM in this system?

The LLM handles natural-language understanding, preference extraction, itinerary explanation, clarification, and high-level planning. It should not own price, availability, payment, authorization, or booking truth.

## 2. Why not let the LLM directly call booking APIs?

Because booking creates irreversible side effects. Tool calls must pass through schema validation, authorization, policy checks, idempotency controls, and often human approval.

## 3. How do you handle price changes?

Search results are treated as provisional. Before checkout, the system fetches the current offer and validates price, availability, restrictions, and expiry. If anything changed, the user receives an updated proposal and must approve again.

## 4. How do you make booking idempotent?

Every booking attempt gets a stable idempotency key. The booking service stores the request and resulting provider reference. Retries first check the idempotency store and reconcile provider state before creating another booking.

## 5. What if the flight booking succeeds but the hotel booking fails?

Use a saga-style workflow. Persist the successful flight booking, execute the hotel step, and if it fails, apply the defined compensation policy: retry, find an alternative, or cancel/refund the already-booked component when appropriate. Do not pretend a distributed transaction exists across suppliers.

## 6. What should be cached?

Static hotel metadata, destination information, policy documents, and other slowly changing data can be cached. Live prices, inventory, and booking status require short TTLs or direct revalidation.

## 7. How would you personalize recommendations?

Combine explicit preferences, current constraints, historical behavior, loyalty information, and contextual signals. Explicit current constraints always override inferred preferences.

## 8. How would you rank flights and hotels?

First apply hard constraints such as dates, budget, traveler count, and direct-flight requirements. Then use a ranking model or weighted score using price, duration, location, flexibility, preference match, supplier quality, and historical acceptance.

## 9. What belongs in RAG?

Policies and explanatory knowledge such as baggage rules, cancellation policies, loyalty documentation, and destination information. Live transactional facts such as price, availability, booking status, and flight status should come from authoritative APIs.

## 10. How do you prevent prompt injection?

Treat provider responses, documents, reviews, and other retrieved content as untrusted data. Keep explicit instruction boundaries, structured tool schemas, tool allow-lists, deterministic authorization, policy validation, and human approval for high-impact actions.

## 11. How do you handle provider outages?

Use timeouts, bounded retries, circuit breakers, provider health scores, bulkheads, and fallback suppliers. Provider-specific errors are normalized into an internal error model so the agent can choose a safe fallback.

## 12. How do you handle long-running workflows?

Use a durable workflow engine or persisted state machine. Search, approval, payment, booking, cancellation, and disruption workflows can survive worker restarts and resume from the last committed state.

## 13. How would you evaluate the agent?

Create a golden dataset of travel requests covering simple searches, ambiguous requests, budget constraints, multi-city trips, booking failures, cancellations, and adversarial prompts. Measure constraint extraction accuracy, tool-call accuracy, task completion, groundedness, unsafe-action rate, latency, and cost.

## 14. What are the most important production metrics?

Track P50/P95/P99 latency, error rate, provider timeout rate, booking success rate, payment success rate, cancellation success rate, search-to-book conversion, tool failure rate, LLM cost, token usage, hallucination/groundedness, and unsafe-action rate.

## 15. How do you handle a flight disruption after booking?

Consume flight-status events, identify affected trips, calculate impact, search eligible alternatives, and present options to the traveler. Rebooking eligibility, fare differences, and refund rules should come from authoritative supplier/policy systems.

## 16. Why separate search from booking services?

Search is high-volume, latency-sensitive, and mostly read-oriented. Booking is lower-volume but correctness-critical and stateful. Separating them allows independent scaling, isolation, and stronger transactional controls.

## 17. What happens if the user changes requirements mid-conversation?

Update the structured constraint state, invalidate affected candidates, rerun only the necessary searches, and generate a new itinerary version. Any material change invalidates an earlier booking approval.

## 18. What if the user says "book the cheapest option"?

The system should define what cheapest means across total cost, baggage, taxes, cancellation conditions, and other relevant constraints. The ranking service provides the factual comparison; the LLM explains the result.

## 19. What would you optimize first if latency is too high?

Trace the complete request. Parallelize independent supplier calls, cache safe static data, reduce unnecessary LLM calls, use a smaller model for extraction/classification, stream responses where appropriate, and optimize provider timeouts. Never remove critical booking validation merely to reduce latency.

## 20. What is the most important architectural principle?

Keep probabilistic reasoning separate from deterministic transaction control:

```text
LLM → understand / plan / explain

Deterministic services → authorize / validate / pay / book / reconcile
```

The system should remain safe even if the model produces an incorrect plan or malformed tool request.
