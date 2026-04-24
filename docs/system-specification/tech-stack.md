# Tech Stack

## Components

| Layer | Technology | Purpose |
|---|---|---|
| **Orchestration** | LangGraph (Python) | Stateful graph-based dialogue management; checkpointing, branching, cycle support |
| **Backend API** | FastAPI | Async, high-performance, native WebSocket support for real-time audio streaming |
| **Speech-to-Text** | Deepgram Nova-3 | Low-latency streaming ASR, multilingual mode |
| **Text-to-Speech** | ElevenLabs / PlayHT | Natural-sounding, low-latency streaming TTS |
| **Telephony** | Twilio Voice | Programmable voice, WebSocket media streams, SMS/WhatsApp API |
| **LLM** | OpenAI GPT-4o / Anthropic Claude | Intent understanding, natural response generation |
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
