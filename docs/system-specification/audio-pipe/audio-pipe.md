# Audio Pipe

    Browser (React/JS LiveKit SDK) / Twilio SIP caller
            ↕ WebRTC / SIP
    LiveKit Cloud/Server  ←→    Agent Process (LiveKit Agents SDK)
                                        ↕
                            Assistant(Agent) + function tools
                                        ↕
                            backend-services (/agent/*) → Supabase

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

Google **Vertex AI** Gemini (`gemini-3.1-flash-lite`, via `livekit-plugins-google` with
`vertexai=True`, location `global`, `thinking_level=minimal`) processes the transcript and generates
the response and tool calls. It replaced the earlier Groq `qwen/qwen3-32b` model (kept only as a
commented-out fallback). Auth is Vertex Application Default Credentials + `GOOGLE_CLOUD_PROJECT`.

### Speech Synthesis

Deepgram **Aura-2** (`aura-2-thalia-en`) converts the LLM output to audio streamed back into the
LiveKit room (EU endpoint). It replaced Cartesia `sonic-3`.

### Noise Cancellation

ai-coustics audio enhancement (`EnhancerModel.SPARROW_S`) is applied to the room audio input via
`room_io.RoomOptions(audio_input=AudioInputOptions(noise_cancellation=...))`. It is applied
uniformly — there is no participant-type branch (no LiveKit Cloud BVC) in the current code.

---
