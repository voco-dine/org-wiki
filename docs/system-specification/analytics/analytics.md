# Analytics

All metrics are derived from structured logs emitted by LangGraph node transitions and stored in NeonDB.

## Metrics

### Call-level

- Order completion rate (confirmed orders / total calls) — primary metric
- Call duration (average, median, per-stage breakdown)
- Abandonment point — which LangGraph node the caller dropped off at
- Human handoff rate
- Average turns per order
- Average order value

### Latency

- Response time per turn (STT + LangGraph processing + TTS)
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
- Abandonment by LangGraph stage (bar chart)

Built with Recharts.
