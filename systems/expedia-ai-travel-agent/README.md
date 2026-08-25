# Expedia AI Travel Agent

A production-grade conversational travel agent inspired by Expedia Group's AI travel experiences. The system helps users discover, compare, plan, book, modify, and manage trips across flights, hotels, activities, transportation, and payments.

> This is a system-design reference architecture, not an implementation of Expedia's proprietary internal systems.

## Problem

Build an AI travel agent that can turn natural-language goals into reliable travel workflows while dealing with live inventory, changing prices, user preferences, booking constraints, payments, cancellations, and travel disruptions.

Example:

> "Plan a 5-day Paris trip from Mumbai for under ₹2 lakh. Prefer direct flights, a 4-star hotel near the city center, and activities suitable for two adults. Show me the best options and book after I approve."

## Core Capabilities

- Conversational trip planning
- Flight and hotel search
- Activity and destination discovery
- Personalized ranking
- Itinerary generation
- Real-time price and availability validation
- Booking and payment orchestration
- Cancellation and modification workflows
- Trip disruption handling
- User preferences and travel history
- Human approval for high-impact actions
- Notifications and trip status updates

## Production Concerns

- Live inventory can change between search and booking
- Prices can change before checkout
- External providers can partially fail
- Booking operations must be idempotent
- Payment and booking state must remain consistent
- Cancellation and refund rules differ by supplier
- Agent actions require authorization and policy checks
- Long-running workflows need durable state
- LLM output must never directly execute side effects
- Personal data and payment data require strict isolation
- Observability must cover both AI quality and transactional reliability

## Real-World Ecosystem

The design can integrate with real travel-provider categories and APIs, including airline/flight distribution providers, hotel inventory providers, maps and places services, payment providers, and notification providers. Provider adapters isolate the agent from supplier-specific APIs and allow fallback or multi-provider search.

## Architecture

See [architecture.md](./architecture.md) for the detailed production architecture and trade-offs.

## Interview Guide

See [mock-interview.md](./mock-interview.md) for senior-level system-design questions and answers.

## Key Design Principle

**The LLM plans and reasons; deterministic services validate availability, price, authorization, policy, payment, and booking state before any irreversible action.**
