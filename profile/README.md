# Jota — Distributed Voice AI Assistant
<img width="1200" height="630" alt="og-image-en" src="https://github.com/user-attachments/assets/bc304728-6f45-460c-a480-06d3a312e600" />

> A modular, privacy-first voice assistant built for local execution. One gateway, six microservices, zero cloud dependency.

Jota is a full-stack AI assistant ecosystem. A physical client (ESP32, web, app) connects to a single entry point — the **gateway** — which resolves identity, loads per-client configuration, and coordinates real-time speech, inference, and synthesis across specialized services.

---

## Architecture

```
 Client (ESP32 / Web / App)
        │  ws://gateway:8004/ws/stream   ← voice session
        │  http://gateway:8004/api/*     ← REST (config, history, models)
        ▼
 ┌─────────────────────────────────────────────────────┐
 │                   jota-gateway (BFF)                 │
 │  Resolves client identity · Loads ClientConfig       │
 │  Propagates settings to every downstream service     │
 └────┬──────────────┬──────────────┬──────────────┬───┘
      │              │              │              │
      ▼              ▼              ▼              ▼
 jota-orchestrator  jota-transcriber  jota-speaker  jota-db
 (LLM + tools)      (STT streaming)   (TTS stream)  (identity + memory)
      │
      ▼
 jota-inference
 (llama.cpp engine)
```

---

## Services

| Repo | Language | Role | Status |
| :--- | :--- | :--- | :--- |
| **[jota-gateway](https://github.com/jota-project/jota-gateway)** | Python | BFF — single entry point, auth, REST API, voice session orchestration | `Stable` |
| **[jota-orchestrator](https://github.com/jota-project/jota-orchestrator)** | Python | LLM routing, tool calling, conversation memory | `Stable` |
| **[jota-inference](https://github.com/jota-project/jota-inference)** | C++ | Local LLM execution via llama.cpp, parallel sessions | `Stable` |
| **[jota-transcriber](https://github.com/jota-project/jota-transcriber)** | C++ | Real-time Whisper STT over WebSocket, partial transcriptions | `Stable` |
| **[jota-speaker](https://github.com/jota-project/jota-speaker)** | Python | Streaming TTS (Kokoro), PCM16 audio output | `Stable` |
| **[jota-db](https://github.com/jota-project/jota-db)** | Python | Identity, per-client config, conversation history, auth | `Stable` |

---

## Key Features

**Single entry point.** All external traffic goes through `jota-gateway`. Services on the internal network are not directly reachable by clients.

**Per-client configuration.** Each client has a `ClientConfig` stored in jota-db — STT language, TTS voice and speed, preferred LLM, system prompt extension, barge-in threshold. The gateway loads it at handshake time and propagates it to every downstream service.

**Real-time voice pipeline.** PCM Float32 mic audio streams to the transcriber, which returns partial transcriptions as they appear. Barge-in interrupts the current TTS turn when a new transcription arrives.

**Tool calling.** The orchestrator supports multi-turn tool calling with a dual-detection parser (streaming + fallback), role-based permissions per client, and MCP (Model Context Protocol) integration for external tools.

**REST API.** The gateway exposes a full REST API for clients to manage their config, browse conversation history, and list available models — no direct database access required.

**Runs 100% locally.** No cloud calls required. The only external network activity is optional (web search tool, Tavily API).

---

## Per-client Configuration

Every client gets its own `ClientConfig`:

| Setting | Controls |
| :--- | :--- |
| `stt_language` | Transcription language sent to jota-transcriber |
| `stt_vad_thold` | Voice activity detection sensitivity |
| `tts_voice` / `tts_speed` | Voice and playback speed for jota-speaker |
| `preferred_model_id` | Which LLM model to use for this client |
| `system_prompt_extra` | Appended to the system prompt on every inference |
| `barge_in_min_chars` | Minimum partial transcription length to trigger barge-in |
| `conversation_memory_limit` | How many messages to include as context |

Managed via `GET/PUT /api/config` and `POST /api/config/reset` on the gateway.

---

## Gateway REST API

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/api/health` | Status of all internal services |
| `GET` | `/api/models` | Available LLM models |
| `GET` | `/api/config` | Client configuration |
| `PUT` | `/api/config` | Update configuration (partial) |
| `POST` | `/api/config/reset` | Reset to defaults |
| `GET` | `/api/conversations` | Conversation history |
| `GET` | `/api/conversations/{id}/messages` | Messages in a conversation |
| `DELETE` | `/api/conversations/{id}` | Archive a conversation |

All endpoints require `X-API-Key` (the client key) except `/api/health`.

---

*Built for local deployment. Designed to grow.*
