# Analytics

Analytics is a module **inside `backend-services`** (`app/services/analytics/`, exposed under
`/api/analytics`), not a separate service. Every metric is a **live SQL aggregate** over the existing
OLTP tables in **Supabase PostgreSQL** — no materialized views, no background jobs, no caching, no
extra dependencies. The dashboard reads it directly.

Metrics are derived from the `call_events` table (the agent streams telemetry, transcript, tool-call
and cart events to `POST /agent/calls/{id}/events`), plus `calls` and `orders`. The funnel "stage"
is derived from the most recent `tool_call` per call — **not** from a LangGraph node, because the
pipeline no longer uses LangGraph (`calls.final_langgraph_node` is left null; see
[ADR-006](../../decisions/006-livekit-over-langgraph.md)).

Every endpoint takes `tenant_id` (required) + optional `store_id` + an optional `from`/`to` window
(defaults to the last 24h) and echoes the resolved `window`. Tenant isolation is enforced in one
place (`analytics/_common.apply_store_scope`, always joining `Store`).

## Implemented endpoints (Phase 1)

| Endpoint | Returns |
|---|---|
| `GET /api/analytics/overview` | `total_calls`, `completion_rate`, `handoff_rate`, `avg_llm_ttft_ms_p50` (the four summary cards) |
| `GET /api/analytics/calls/timeseries` | calls bucketed by hour/day (`points[]` of `{ts, count}`) |
| `GET /api/analytics/calls/last-stage` | funnel: distribution of the last `tool_call` bucket per call (`stages[]`) |
| `GET /api/analytics/calls/latency` | pipeline latency `p50`/`p95` per stage or per model, over `stt/llm/tts/eou` metric events |
| `GET /api/analytics/orders/top-items` | top-N items by quantity (`{item_id, item_name, quantity, order_count}`) |
| `GET /api/analytics/orders/confused-items` | items the agent failed to resolve (`tool_call` events with `status ∈ {ambiguous, off_menu, not_in_order}`) |
| `GET /api/analytics/conversation/turns` | `avg_turns`, `agent_turns`, `interrupted_agent_turns`, `interruption_rate` |

## Deferred (not implemented)

Gated on upstream data capture:

- **Sync-service health** (`/api/analytics/sync/health`) — Phase 2, waiting on `sync-service` to emit telemetry.
- **Feedback summary / average rating** (`/api/analytics/feedback/summary`) — waiting on the WhatsApp/SMS feedback capture pipeline ([ADR-005](../../decisions/005-whatsapp-sms-feedback.md)). The overview's fourth card was therefore changed from "average rating" to LLM time-to-first-token p50.
- **Peak hours / average order value** — derivable from the current tables but not yet exposed as endpoints.

See `backend-services/docs/ANALYTICS_PROPOSAL.md` for the full design and phasing.

## Dashboard

The `dashboard-frontend` (Next.js 15 / React 19) reads these endpoints:

- Four summary cards: completion rate, LLM TTFT p50, average latency, handoff rate
- Calls over time (line chart, `/calls/timeseries`)
- Funnel / last-stage distribution (bar chart, `/calls/last-stage`)
- Latency percentiles per stage (`/calls/latency`)

> The `/api/analytics` and `/api/calls` surfaces are currently **unauthenticated** (prototype scale).
> Guard them before any external exposure.
