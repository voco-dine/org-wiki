# ADR-006: LiveKit Agents SDK over LangGraph

**Status:** Accepted  
**Date:** 2026-05

## Decision

Use the **LiveKit Agents SDK** as the sole orchestration and voice pipeline framework instead of building a custom LangGraph state machine on top of LiveKit.

## Context

[ADR-001](001-langgraph-over-platforms.md) decided to build custom orchestration with LangGraph rather than adopting a managed low-code voice platform. The core requirements were: stateful multi-phase conversation control, structural menu validation, full latency control, and data ownership.

When implementation began, it became clear that the LiveKit Agents SDK already satisfies all of those requirements natively:

- **Handoffs and tasks** replace explicit LangGraph graph nodes — each conversation phase is an `Agent` subclass or a scoped task, with the same structured transitions.
- **`@function_tool` methods** provide structural cart validation; the LLM calls them to mutate state, not to decide whether validation runs.
- **The full voice pipeline** (VAD → STT → turn detection → LLM → TTS) is owned end-to-end by LiveKit Agents, giving the same latency control.
- All call data flows to our own NeonDB via `backend-services`, satisfying data ownership.

Adding LangGraph on top would have been a redundant orchestration layer with no incremental benefit.

## Rationale

- The motivations in ADR-001 are preserved — we are still on a custom stack, not a managed platform.
- LiveKit Agents' handoffs/tasks pattern covers the same multi-phase conversation control that LangGraph's state graph was chosen for.
- One fewer dependency reduces operational surface area.
- The LiveKit Agents SDK is actively maintained with a stable Python API and first-class testing/eval support.

## Trade-offs Accepted

- Conversation flow is expressed through LiveKit's handoff/task primitives rather than a visual graph — less visual tooling for state inspection.
- Rollback to an earlier conversation state (e.g. "change the first item") must be managed through agent instructions and tool logic rather than LangGraph checkpointing.
