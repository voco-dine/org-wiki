# Tech Stack

## Components

The table reflects what runs in code today. Where it differs from the original design intent, the
"Purpose" note says so.

| Layer | Technology | Purpose |
|---|---|---|
| **Orchestration** | LiveKit Agents SDK (Python) | Single prompt-driven `Assistant(Agent)` + function tools. Replaced LangGraph — see [ADR-006](../decisions/006-livekit-over-langgraph.md) |
| **Backend API** | FastAPI + async SQLAlchemy 2 | `backend-services`: `/agent/*` (agent), `/api/calls` + `/api/analytics` (dashboard) |
| **Transport / Telephony** | LiveKit Cloud (WebRTC + SIP) | Media transport and agent dispatch; PSTN via a Twilio SIP trunk (`twilio_call_sid` recorded per call) |
| **Speech-to-Text** | Deepgram `nova-3` | Low-latency streaming ASR (EU endpoint) |
| **LLM** | Google **Vertex AI** Gemini (`gemini-3.1-flash-lite`) | Intent understanding + response generation, via `livekit-plugins-google`. Migrated from Groq `qwen3-32b` |
| **Text-to-Speech** | Deepgram Aura-2 (`aura-2-thalia-en`) | Streaming TTS. Migrated from Cartesia `sonic-3` |
| **VAD / Turn / Noise** | Silero VAD · `MultilingualModel` turn detector · ai-coustics `SPARROW_S` | End-of-turn detection and audio enhancement |
| **Menu cache** | Redis (*planned, not implemented*) | [ADR-004](../decisions/004-redis-menu-cache.md) intent. Today the menu is cached in agent process memory per call |
| **Persistent Storage** | PostgreSQL (**Supabase**) | Source of truth: tenants, stores, menu, call logs/events, orders, feedback |
| **POS Integration** | `sync-service` (raw asyncpg) + adapter | Menu/order sync between POS and the DB — see [ADR-003](../decisions/003-adapter-layer.md) |
| **Dashboard** | Next.js 15 / React 19 + LiveKit SDK | Control panel + live monitoring; reads `/api/calls` and `/api/analytics` |

---

## Latency Budget

Target: **< 1 second** end-to-end perceived response time per turn.

| Component | Budget | Notes |
|---|---|---|
| ASR (Deepgram streaming) | ~200 ms | Streaming partial transcripts |
| LLM inference | ~300–500 ms | Including tool calls where needed |
| Menu / cache lookup | ~1 ms | Redis, in critical path |
| TTS (streaming) | ~200 ms | Time to first audio byte |
| **Total** | **~700–900 ms** | Within budget on the happy path |

Adding a mandatory vector DB retrieval hop would push this to ~1.1–1.4 s, exceeding the target. See [ADR-002](../decisions/002-tiered-menu-context.md).
