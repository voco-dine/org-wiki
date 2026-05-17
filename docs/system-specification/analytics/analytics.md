# Analytics

All metrics are derived from structured `CallEvent` records emitted by the agent during a session and persisted to NeonDB via `backend-services`.

## Metrics

### Call-level

- Order completion rate (confirmed orders / total calls) — primary metric
- Call duration (average, median, per-stage breakdown)
- Abandonment point — which conversation phase the caller dropped off at
- Human handoff rate
- Average turns per order
- Average order value

### Latency

- Response time per turn (STT + LLM + TTS)
- Time to first audio byte (TTS)

### Menu / Order

- Most ordered items
- Most confused menu items (high correction rate)
- Invalid order attempts — how often the agent catches an off-menu request
- Peak hours

## Dashboard

Single-page React app reading from NeonDB via FastAPI endpoints.

- Four summary cards: completion rate, average rating, average latency, handoff rate
- Calls over time (line chart)
- Abandonment by conversation phase (bar chart)

Built with Recharts.
