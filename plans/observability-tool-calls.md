# Frontend: surface tool calls (and their inputs) per call in the dashboard

**Repo:** `dashboard-frontend`
**Type:** Feature / observability
**Status:** Backlog (future scope)
**Created:** 2026-06-25
**Depends on:** backend-services `GET /api/calls/{call_id}/events` (shipped — see below)

## Context

We're building observability for the voice agent. The voice agent already emits a
`tool_call` event for every function-tool invocation (one per call to `add_item`,
`remove_item`, `modify_item`, `get_item_details`, `get_category_items`,
`get_order_summary`, `confirm_order`, `request_handoff`), including the inputs the
tool was called with. These are persisted to `call_events` in backend-services.

backend-services now exposes a read endpoint for them. The frontend doesn't surface
any of this yet — the call-detail view shows orders only. This ticket covers adding a
view so we can see, per call, which tools the agent invoked and with what inputs.

## Backend contract (already available)

```
GET /api/calls/{call_id}/events?event_type=tool_call&limit=200&offset=0
```

- `event_type` — optional filter; pass `tool_call` for tool calls. Omit to get the
  full event timeline (also includes `transcript_user`/`transcript_agent`,
  `stt_metrics`/`llm_metrics`/`tts_metrics`/`eou_metrics`, `item_added`/`item_removed`/
  `item_modified`, `handoff_requested`, `error`).
- `limit` 1–1000 (default 200), `offset` >= 0. Events are returned **oldest-first**
  (chronological).
- 404 if the call doesn't exist; `[]` if the call has no matching events.
- No auth (consistent with the rest of `/api/*` at prototype scale).

Response is a JSON array of events:

```jsonc
[
  {
    "id": "…uuid…",
    "call_id": "…uuid…",
    "event_type": "tool_call",
    "node_name": "add_item",            // the tool name
    "payload": {
      "args":   { "item_name": "margherita", "quantity": 1, "modifiers": [] }, // the inputs
      "status": "added",                // tool-reported outcome
      "error":  null                    // present only on failure
    },
    "latency_ms": 123,
    "occurred_at": "2026-06-25T10:00:00Z"
  }
]
```

For a tool-call row, render `node_name` as the tool and `payload.args` as the inputs.
`payload.status`, `payload.error`, and `latency_ms` are useful secondary columns.

## Scope

- In the call-detail view, add a "Tool calls" section/tab that fetches
  `GET /api/calls/{call_id}/events?event_type=tool_call` and lists each tool call:
  tool name, inputs (`payload.args`), status, latency, timestamp.
- Render inputs readably (key/value or pretty-printed JSON), not a raw blob.
- Handle empty (`[]` → "No tool calls for this call") and error/404 states.
- Wire the API base URL through the existing config the way other backend reads are.

## Acceptance criteria

- [ ] Opening a call shows its tool calls in chronological order with inputs visible.
- [ ] A call with no tool calls renders a clear empty state, not an error.
- [ ] Latency and status are shown per tool call; failed calls (`error` set) are visually distinct.
- [ ] No regressions to the existing orders view.

## Out of scope / future

- Full event timeline view (transcripts + pipeline metrics interleaved) — separate ticket.
- Cross-call aggregation ("most-used tools", "tools that most often error") — belongs in
  a backend analytics aggregate (`app/services/analytics/`), not this per-call view.
- Realtime/live streaming of events during an in-progress call.

## References

- Backend route: `backend-services/app/api/routes/calls.py` (`list_call_events`)
- Backend service: `backend-services/app/services/calls.py` (`list_call_events`)
- Response schema: `backend-services/app/schemas/agent.py` (`CallEventRead`)
- Event taxonomy: `backend-services/app/services/agent_core/event_types.py`
- Producer (voice agent): `voice-agent-backend/src/telemetry/tracing.py` (`traced_tool`),
  `voice-agent-backend/src/telemetry/client.py` (`trace_tool`)
