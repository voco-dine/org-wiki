# Tech Stack

## Components

| Layer | Technology | Purpose |
|---|---|---|
| **Voice Pipeline** | LiveKit Agents SDK (Python) | Agent/AgentSession abstraction over STT → LLM → TTS; native handoffs and tasks for multi-phase workflows |
| **Backend API** | FastAPI | Async, high-performance REST API for orders, menu, call events |
| **Speech-to-Text** | Deepgram Nova-3 | Low-latency streaming ASR, multilingual mode |
| **Text-to-Speech** | Cartesia Sonic-3 | Low-latency streaming TTS |
| **Telephony** | LiveKit Cloud (WebRTC + SIP) | WebRTC media transport; SIP trunk for inbound phone calls; SMS/WhatsApp via Twilio (ADR-005) |
| **LLM** | Groq (qwen3-32b) | Intent understanding, cart tool calls, natural response generation |
| **Menu Read Cache** | Redis | Per-call menu cache; falls back to NeonDB on miss or failure |
| **Persistent Storage** | PostgreSQL (NeonDB) | Source of truth: menu data, call logs, analytics, order history, feedback |
| **POS Integration** | REST APIs (direct) | Structured order JSON, written directly to restaurant's POS |
| **Dashboard** | React + Recharts | Single-page analytics dashboard |

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
