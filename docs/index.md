# VocoDine Wiki

!!! warning "Early Development"
    This project is in early development. Architecture and implementation details are subject to change. Documentation reflects current design intent, not a finished system.

Welcome to the VocoDine organization wiki. This site documents the design, architecture, and operational details of the VocoDine system.

VocoDine is an AI-powered voice agent that handles restaurant phone orders autonomously through natural, multi-turn conversation. The agent takes the call, greets the customer, processes their order (items, modifiers, special requests), confirms it verbally and via SMS/WhatsApp summary, then pushes a structured order directly to the restaurant's POS system.

## Sections

- [Architecture](system-specification/architecture.md) — System diagram, call flow, LangGraph state graph, and menu context strategy.
- [Tech Stack](system-specification/tech-stack.md) — Technology choices and latency budget.
- [Audio Pipe](system-specification/audio-pipe/audio-pipe.md) — Voice pipeline components (STT, LLM, TTS, VAD).
- [Analytics](system-specification/analytics/analytics.md) — Metrics and dashboard.
- [Decisions](decisions/index.md) — Architecture Decision Records.
- [Scope](scope.md) — What is and isn't in scope, plus timeline.
- [Evaluation](evaluation.md) — How the system will be evaluated.
