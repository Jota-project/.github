# Jota — a personal agent with a voice

<img width="1200" height="630" alt="og-image-en" src="https://github.com/user-attachments/assets/bc304728-6f45-460c-a480-06d3a312e600" />

Jota is a personal agent you talk to. It remembers what you tell it and does things for you, at home and day to day.

**Website:** [jota-project.com](https://jota-project.com)

This organization hosts the open-source pieces that carry your voice in and out: the gateway (Jota-Gateway), speech-to-text, text-to-speech, a voice client and a wake-word trainer. The reasoning, memory, agents and tools live in [OpenClaw](https://github.com/openclaw/openclaw), an open-source AI orchestrator.

## How it fits together

```
 Client (Jota-Screen · jota-voice · iPhone app)
    │  WebSocket: mic audio up, text + audio down
    ▼
 jota-gateway  ── single entry point: client keys, sessions, agent routing
    │
    ├─► Speech-to-text:  jota-transcriber (whisper.cpp)  or ElevenLabs
    ├─► Orchestrator:    OpenClaw (reasoning, memory, agents, tools)
    └─► Text-to-speech:  jota-speaker (Kokoro)            or ElevenLabs
    │
    ▼
 Client  ◄─ streamed text and audio
```

Clients only ever talk to the gateway. Speech providers are chosen by whoever runs the gateway and can be swapped without touching any client.

## Repositories

| Repo | Status | What it is |
| :--- | :--- | :--- |
| [jota-gateway](https://github.com/jota-project/jota-gateway) | Active | Jota-Gateway: BFF and single entry point (Python, FastAPI). Client WebSocket, admin API, OpenAI-compatible API for Home Assistant. |
| [jota-transcriber](https://github.com/jota-project/jota-transcriber) | Active | Streaming speech-to-text server on whisper.cpp (C++, CUDA). |
| [jota-speaker](https://github.com/jota-project/jota-speaker) | Active | Streaming text-to-speech on Kokoro ONNX (Python). Also speaks Wyoming for Home Assistant. |
| [jota-voice](https://github.com/jota-project/jota-voice) | Active | Open-source voice client for macOS and Android (Termux), with on-device wake word (openWakeWord). |
| [jota-wake-trainer](https://github.com/jota-project/jota-wake-trainer) | Active | Trains custom wake words for openWakeWord. |
| [jota-orchestrator](https://github.com/jota-project/jota-orchestrator) | Legacy (archived) | Former in-house orchestrator. Replaced by OpenClaw. |
| [jota-inference](https://github.com/jota-project/jota-inference) | Legacy (archived) | Former llama.cpp inference server used by jota-orchestrator. Replaced by OpenClaw. |
| [jota-db](https://github.com/jota-project/jota-db) | Legacy (archived) | Former identity and config service. Deprecated since jota-gateway v1.9.0, which keeps its own database. |

Legacy repos are archived: read-only, kept for reference. New setups should not depend on them.

## Clients

- **jota-voice** — open source, macOS and Android. Runs on Linux too, installed by hand for now.
- **Jota-Screen** — for the TV. Closed source.
- **Jota for iPhone** — early stage. Closed source.

Want to build your own client? The gateway protocol is documented in [`jota-gateway/docs/client-protocol.md`](https://github.com/jota-project/jota-gateway/blob/main/docs/client-protocol.md). Audio goes up as PCM Float32 at 16 kHz and comes back as PCM16 at 24 kHz.

## Where it runs

Jota is installed locally: the gateway, speech and orchestrator run on machines of yours. What leaves your home depends only on the external providers you choose, such as ElevenLabs for speech, a cloud language model for OpenClaw, or a tunnel or proxy to reach the gateway from outside. Their terms, and what they do with your data, are between you and them.

## Key docs

- [Client protocol](https://github.com/jota-project/jota-gateway/blob/main/docs/client-protocol.md) — handshake, turns, audio frames, barge-in, errors
- [jota-gateway README](https://github.com/jota-project/jota-gateway#readme) — endpoints, configuration, speech providers
- [jota-transcriber API guide](https://github.com/jota-project/jota-transcriber/blob/main/clients/API_GUIDE.md)
- [jota-speaker README](https://github.com/jota-project/jota-speaker#readme) — WebSocket, OpenAI-compatible and Wyoming surfaces
- [Contributing](https://github.com/jota-project/.github/blob/main/CONTRIBUTING.md) and [standards](https://github.com/jota-project/.github/blob/main/STANDARDS.md)
