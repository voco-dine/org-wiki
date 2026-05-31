# ADR-006: LiveKit Agents SDK over LangGraph for Orchestration

**Status:** Accepted  
**Date:** 2026-05  
**Supersedes:** the orchestration-framework portion of [ADR-001](001-langgraph-over-platforms.md)

## Decision

Run conversational orchestration as a **single prompt-driven LiveKit `Agent`** (the
`Assistant(Agent)` in `voice-agent-backend/src/assistant.py`) exposing function tools, instead of a
**LangGraph** state machine. LangGraph (and LangChain) were removed from both `voice-agent-backend`
and `backend-services`.

## Context

[ADR-001](001-langgraph-over-platforms.md) chose LangGraph for stateful, graph-based dialogue
control (Greeting → Intent → Selection → Modifiers → Disambiguation → Upselling → Confirmation →
Handoff, with checkpointing and rollback). In practice the voice pipeline runs on the **LiveKit
Agents SDK** — LiveKit Cloud already owns the transport, the audio/turn loop, VAD, turn detection,
and the STT/LLM/TTS plugin chain. Layering a separate LangGraph state machine on top of that loop
duplicated state, added latency and moving parts, and never earned its keep: the conversation is
handled naturally by one agent that calls tools.

## Rationale

- **One runtime, not two.** LiveKit Agents already drives the turn loop; a second orchestration graph
  was redundant state to keep in sync.
- **Lower latency / fewer hops.** The LLM decides flow and calls tools directly; there is no graph
  transition layer between STT and the response.
- **Menu grounding stays structural.** It does not depend on a mandatory graph node — `add_item`
  resolves against the fetched menu and returns `off_menu` / `ambiguous` (with candidates) rather than
  adding an item that doesn't exist. Disambiguation and modifier validation are tool-return contracts.
- **Simpler to test and maintain.** A system prompt plus eight tools (`add_item`, `remove_item`,
  `modify_item`, `get_item_details`, `get_category_items`, `get_order_summary`, `confirm_order`,
  `request_handoff`) is easier to unit-test (TDD with the LiveKit Agents eval framework) than graph
  nodes, edges, and guards.
- **Fewer dependencies.** `langgraph` / `langchain*` were dropped from both Python repos; the
  backend `agent_core/graph.py`, `nodes.py`, `edges.py`, `state.py`, `model_config.py` were deleted.

## Trade-offs Accepted

- **No explicit checkpointing or state rollback.** Flow correctness now rests on prompt design and
  tool-return statuses, backed by tests, rather than on graph guards. Mid-conversation corrections
  ("change the first item") are handled by `modify_item` / `remove_item`, not graph cycle-back.
- **Analytics "stage" is reinterpreted.** The funnel/last-stage metric is derived from the most
  recent `tool_call` per call instead of a graph node. The `calls.final_langgraph_node` column is
  retained (now left null) for backward compatibility; see
  [analytics](../system-specification/analytics/analytics.md).
- **ADR-001's stateful-graph rationale no longer applies** to this layer. The broader
  "custom stack over a low-code platform" decision in ADR-001 still holds — this only changes *how*
  the custom stack is orchestrated.

## See also

- [Architecture → Conversation orchestration](../system-specification/architecture.md)
- [Sequence diagrams](../system-specification/sequence.md)
- [ADR-002: Tiered menu context](002-tiered-menu-context.md) (still in force)
