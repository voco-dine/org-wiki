# ADR-003: Adapter Layer (Ports & Adapters)

**Status:** Accepted  
**Date:** 2026-04

## Decision

Build a formal adapter layer that normalises any restaurant's data source into a standard internal schema before it reaches the agent or the database.

## Context

Different restaurants use different POS systems, data formats, and integration capabilities. The agent needs a consistent internal representation regardless of the source.

## Rationale

- Adding a new restaurant means writing an adapter, not modifying the agent or the database schema.
- A broken adapter (schema mismatch, upstream API failure) is isolated — it doesn't crash a live call.
- The adapter is the boundary between the restaurant's system and ours, which is the foundation of the white-label model.

**Two integration modes:**

- **Push (restaurant integrates with us).** We expose a standardised API. The restaurant or POS vendor pushes menu updates and availability changes; we send orders back via webhook. Cleaner and real-time.
- **Pull (we integrate with them).** Our adapter polls their existing API or export and normalises it. Fallback for restaurants that won't build an integration. This mode will be built first for the FYP.

**Read/write separation.** The adapter handles reads (menu, availability, pricing). Order placement (write) bypasses the adapter and goes directly to the restaurant's POS API — no intermediate layer that could create consistency issues. See [ADR — Direct POS Write Path](003-adapter-layer.md) below.

## Direct POS Write Path

Confirmed orders are sent directly to the restaurant's POS API without passing through the adapter or cache. If the POS rejects an order, the agent knows immediately and can inform the customer. There is no async sync step that could hide failures.

## Trade-offs Accepted

- Adapter must be written per restaurant or per POS system — upfront integration cost per new customer.
- Pull mode requires reverse-engineering or accessing the restaurant's existing data export format.
- Push mode requires the restaurant to build against our API spec, which may not be feasible for all partners.
