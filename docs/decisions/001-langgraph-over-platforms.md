# ADR-001: LangGraph over Low-Code Platforms

**Status:** Accepted — **orchestration framework superseded by [ADR-006](006-livekit-over-langgraph.md)** (2026-05)  
**Date:** 2026-04

> **Update (2026-05):** The "build custom instead of a low-code platform" decision below still
> stands. What changed is the orchestration framework — LangGraph was dropped in favour of the
> **LiveKit Agents SDK** (a single prompt-driven agent with function tools); see ADR-006. Also note
> the production database is **Supabase PostgreSQL**, not NeonDB as the original rationale says.
> Read the rest of this record as the point-in-time reasoning, not the current implementation.

## Decision

Build custom orchestration with LangGraph (Python) instead of using low-code voice AI platforms such as Vapi, Bland, Retell, n8n, or GHL.

## Context

Several managed voice AI platforms exist that provide pre-built telephony, STT, LLM, and TTS pipelines. The question was whether to use one of these or build a custom stack.

## Rationale

- **Stateful conversation control.** Low-code tools treat each LLM call as mostly stateless. LangGraph provides an explicit state machine where each phase (item selection, modifier handling, disambiguation, rollback) is a defined graph transition with guards — not prompt engineering.
- **Architectural menu grounding.** On low-code platforms, the LLM decides whether to call a validation function. In our setup, menu validation is a mandatory graph node — the LLM cannot confirm an item that doesn't exist. Enforcement is structural, not prompt-level.
- **Latency control.** We control the full streaming chain and can implement fast-path deterministic responses, response caching, and interrupt handling. Black-box platforms don't expose this.
- **Data ownership.** All call data, analytics, and conversation logs live in our own NeonDB. Required for the white-label model.
- **Cost at scale.** Custom stack costs ~$0.03/min (Deepgram + TTS + Twilio + LLM) vs. $0.05–0.10/min on managed platforms.

## Trade-offs Accepted

- Slower time to first prototype (weeks vs. hours on a managed platform).
- We own infrastructure, scaling, and failover.
- Telephony edge cases (hold music detection, voicemail, carrier quirks) must be discovered and handled ourselves.
