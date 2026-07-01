# Jota — Distributed Voice AI Assistant

> A modular, privacy-first voice assistant built for local execution. One gateway, specialized microservices, OpenAI-compatible.

Jota is a full-stack AI assistant ecosystem. A physical client (ESP32, web, app, Termux, Home Assistant) connects to a single entry point — the **gateway** — which resolves identity, loads per-client configuration, and coordinates real-time speech, LLM, and synthesis across specialized services.

---

## Services

| Repo | Language | Role | Status |
| :--- | :--- | :--- | :--- |
| **[jota-gateway](https://github.com/Jota-project/jota-gateway)** | Python | BFF — single entry point, auth, REST API, voice session orchestration | `Maintained` |
| **[jota-transcriber](https://github.com/Jota-project/jota-transcriber)** | C++17 | Real-time Whisper STT over WebSocket | `Maintained` |
| **[jota-speaker](https://github.com/Jota-project/jota-speaker)** | Python | Streaming TTS (Kokoro) + Wyoming protocol for Home Assistant | `Maintained` |
| **[jota-orchestrator](https://github.com/Jota-project/jota-orchestrator)** | Python | LLM routing, tool calling, conversation memory | `Alternative` |
| **[jota-inference](https://github.com/Jota-project/jota-inference)** | C++ | Local LLM execution via llama.cpp | `Alternative` |
| **[jota-db](https://github.com/Jota-project/jota-db)** | Python | Centralized auth for legacy service setups | `Deprecated` |

### Alternative vs Maintained

The **Maintained** path is the recommended one — the gateway uses local SQLite for identity and configuration, and **OpenClaw** as the LLM orchestrator (external open-source project).

The **Alternative** path (`jota-orchestrator` + `jota-inference`) keeps Jota 100% self-hosted with first-party Python/C++ services. It receives less active development; new deployments should prefer the Maintained path unless full local control of the inference layer is a hard requirement.

The **Deprecated** path (`jota-db`) is kept for compatibility with setups that still use it as a centralized auth backend. New code should use per-service `AUTH_TOKEN` or each service's own storage.

See [`ARCHITECTURE.md`](ARCHITECTURE.md) for the full architecture map and [`TASKS.md`](TASKS.md) for active work.

---

## Clients

| Repo | Description |
| :--- | :--- |
| **[jota-voice](https://github.com/Jota-project/jota-voice)** | Termux/Android client. Replaces wyoming-satellite with direct WS streaming to the gateway. |
| **[jota-display](https://github.com/Jota-project/jota-display)** | Always-on kiosk UI (Vue 3 + SSE) showing voice session state and home controls. |

Both clients run on the same Android device and connect to the gateway over LAN.

### Personal experiments

The author also maintains auxiliary tooling outside this organization. These are not part of the Jota stack but may be useful:

- [SitoSt/jota-wake-trainer](https://github.com/SitoSt/jota-wake-trainer) — CLI to train custom wake words with openWakeWord.
- [SitoSt/jota-dashboard](https://github.com/SitoSt/jota-dashboard) — Web dashboard.

---

## Quick start

Clone the maintained repositories:

```bash
git clone https://github.com/Jota-project/jota-gateway.git
git clone https://github.com/Jota-project/jota-transcriber.git
git clone https://github.com/Jota-project/jota-speaker.git
# Plus OpenClaw running separately (see OpenClaw docs)
# Plus a client: jota-voice or jota-display
```

Then follow each repo's README for environment variables and Docker compose instructions. The gateway README is the entry point.

---

*Built for local deployment. Designed to grow.*
