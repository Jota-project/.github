# Jota — System architecture

> Last reviewed: 2026-10-26, against jota-gateway v1.21.0, jota-transcriber v1.4.2, jota-speaker v1.1.3 and OpenClaw 2026.10.5.
>
> This page is a map. Wire protocols and configuration live in each repo, which is the source of truth; follow the links instead of copying details here.

## Overview

Jota separates the voice from the brain. Clients capture and play audio. **Jota-Gateway** (repo `jota-gateway`) is the only thing they connect to. Behind it, speech-to-text, an orchestrator and text-to-speech do the work.

```
 Client (Jota-Screen · jota-voice · iPhone app)
    │  ws://<gateway>/ws/stream
    │  handshake with a per-client client_key
    ▼
 jota-gateway  (Python/FastAPI, :8004)
    │  identity, per-client config and agent routing in its own SQLite database
    │
    ├─► STT: jota-transcriber (WebSocket /api/stt)  ·  or ElevenLabs Scribe
    ├─► Orchestrator: OpenClaw (WebSocket, default 127.0.0.1:18789)
    └─► TTS: jota-speaker (WebSocket /ws, :8005)     ·  or ElevenLabs
```

## Components

| Component | Role | Tech | Docs |
|---|---|---|---|
| jota-gateway (Jota-Gateway) | Single entry point. Client keys, sessions, agent routing, coordination of STT, orchestrator and TTS. Admin API and an OpenAI-compatible API for Home Assistant. | Python, FastAPI | [README](https://github.com/jota-project/jota-gateway#readme), [client protocol](https://github.com/jota-project/jota-gateway/blob/main/docs/client-protocol.md), [OpenClaw protocol](https://github.com/jota-project/jota-gateway/blob/main/docs/openclaw-protocol.md) |
| jota-transcriber | Streaming speech-to-text with partial and final transcriptions. Also `POST /v1/audio/transcriptions`. | C++, whisper.cpp, CUDA | [README](https://github.com/jota-project/jota-transcriber#readme), [API guide](https://github.com/jota-project/jota-transcriber/blob/main/clients/API_GUIDE.md) |
| jota-speaker | Streaming text-to-speech, sentence by sentence. Also OpenAI-compatible `POST /v1/audio/speech` and a Wyoming server for Home Assistant. | Python, Kokoro ONNX | [README](https://github.com/jota-project/jota-speaker#readme) |
| OpenClaw | Orchestrator: reasoning, memory, agents, tools, scheduled tasks. External project. | Node.js, TypeScript | [openclaw/openclaw](https://github.com/openclaw/openclaw) |
| jota-voice | Open-source voice client for macOS and Android (Termux). On-device wake word with openWakeWord. | Python | [README](https://github.com/jota-project/jota-voice#readme) |
| jota-wake-trainer | Trains custom wake-word models for openWakeWord. | Python | [README](https://github.com/jota-project/jota-wake-trainer#readme) |

Jota-Screen (TV) and the iPhone app are closed source. They use the same client protocol as jota-voice.

## A voice turn

1. The client detects the wake word on the device and opens a WebSocket session with the gateway, sending its `client_key`.
2. The gateway validates the key against its own database and loads that client's config (STT language, TTS voice and speed, barge-in, default and allowed agents, output modes). Revoked or deactivated clients are disconnected within about 10 s.
3. Mic audio (PCM Float32, 16 kHz) is forwarded to the STT provider. Partial transcriptions stream back to the client. The final transcription goes back to the client, which confirms it with a `send` message.
4. The gateway sends the text to the client's agent in OpenClaw.
5. The reply streams back as text tokens. The gateway forwards them to the client and to the TTS provider at the same time, and sends the audio (PCM16, 24 kHz, binary frames with a turn header) on the same WebSocket.
6. Barge-in, cancellation and agent-initiated (push) turns are described in the [client protocol](https://github.com/jota-project/jota-gateway/blob/main/docs/client-protocol.md).

The STT and TTS providers (`jota` or `elevenlabs`) are chosen by whoever runs the gateway, for all its clients, via `STT_PROVIDER` / `TTS_PROVIDER` or `PUT /admin/speech`. Clients see the same messages and audio formats either way.

## Deployments

- **Jota Cloud**: gateway, speech and orchestrator run on Jota's servers. Only the client runs at home.
- **Jota Local**: everything runs on a machine of yours. Audio stays on your network if you use jota-transcriber and jota-speaker; the language model used by OpenClaw and any tunnel you use to reach the gateway from outside are separate choices.

## Home Assistant

- jota-speaker as a TTS engine, through Wyoming (default port 20424).
- jota-gateway as a conversation agent, through its OpenAI-compatible API (`/v1/chat/completions`).

## Legacy repositories (archived)

These repos are no longer part of the architecture. They are archived (read-only) and kept for reference only.

| Repo | Replaced by |
|---|---|
| jota-orchestrator · Legacy (archived) | OpenClaw |
| jota-inference · Legacy (archived) | OpenClaw, with the language model it is configured to use |
| jota-db · Legacy (archived) | jota-gateway's own database (since v1.9.0). jota-transcriber and jota-speaker can still use it as an auth backend; new setups don't need it. |
