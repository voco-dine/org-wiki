# ADR-005: WhatsApp / SMS Post-Call Feedback

**Status:** Planned  
**Date:** 2026-04

## Decision

Collect post-call feedback via WhatsApp (or SMS fallback) using the existing Twilio integration. Responses are logged to NeonDB.

## Context

The evaluation framework requires a user satisfaction signal. A post-call survey needs to be low-friction for the customer and easy to implement given the existing telephony stack.

## Planned Flow

1. Order confirmed → SMS/WhatsApp order summary already sent as part of the confirmation step.
2. ~30–45 minutes after the order (after estimated delivery), send a follow-up: *"How was your ordering experience? Reply 1–5."*
3. Rating ≤ 3 → follow up: *"Sorry to hear that. What went wrong? Reply briefly."* Store free-text.
4. Rating 4–5 → reply *"Thank you!"* and close.
5. Log rating + caller ID + order ID to NeonDB.

## Rationale

- Piggybacks on the existing Twilio WhatsApp integration — no new external dependency.
- WhatsApp messages have high open rates; low friction for the customer.
- Produces both a quantitative signal (1–5 rating) and qualitative signal (free-text for low ratings).
- Feeds directly into the [evaluation framework](../evaluation.md).

## Trade-offs Accepted

- Feedback window is delayed (~30–45 min) and may not capture in-call experience accurately.
- Requires the customer's phone number to be reachable on WhatsApp; SMS fallback needed for those without it.
- Free-text responses are unstructured and require manual review.
