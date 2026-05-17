# Audio Pipe

    Browser (React/JS LiveKit SDK)
            ↕ WebRTC
    LiveKit Cloud/Server  ←→    Agent Process (LiveKit Agents SDK)
                                        ↕
                            Agent / AgentSession / @function_tool
                                        ↕
                            backend-services (menu, orders, events)

---

## Core components

The voice pipeline follows a sequential STT → LLM → TTS architecture, orchestrated by LiveKit Agents.

### Voice Activity Detection

Silero VAD is preloaded during worker initialization via the `prewarm` hook, ensuring the model is warm before any user session begins and avoiding cold-start latency on the first connection.

### Speech Recognition

Deepgram Nova-3 handles speech-to-text with multilingual mode enabled, allowing the agent to handle mixed-language input without requiring explicit language configuration per session.

### Turn Detection

A `MultilingualModel` turn detector is used rather than relying on VAD alone. VAD only detects silence; the multilingual model uses a transformer to semantically determine when the user has completed their turn, reducing false end-of-turn triggers mid-sentence.

### Language Model

Groq (qwen3-32b) processes the transcript and generates a response. `preemptive_generation` is enabled, meaning the LLM begins generating a response before turn detection fully confirms end-of-turn — shaving meaningful latency off perceived response time.

### Speech Synthesis

Cartesia Sonic-3 converts the LLM output to audio streamed back into the LiveKit room.

### Noise Cancellation

Audio enhancement is applied conditionally based on participant type. SIP callers receive BVC Telephony noise cancellation, optimized for phone audio characteristics. Browser participants receive ai_coustics enhancement instead, tuned for WebRTC audio profiles.

---
