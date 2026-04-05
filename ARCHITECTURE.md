# Jota — Arquitectura del Sistema

> Última actualización: 2026-04-05

---

## 1. Visión General

Jota es un asistente de voz distribuido. Un cliente físico (ESP32, app, web) habla con un único punto de entrada (gateway), que orquesta varios microservicios especializados.

### Mapa de servicios

```
Physical Client (ESP32 / Web / App)
        │  WebSocket  ws://gateway:8004/ws/stream
        │  REST HTTP  http://gateway:8004/api/*
        │  Handshake: { client_key, input_mode, output_mode, ... }
        ▼
  [jota-gateway]  — Python/FastAPI — Puerto 8004
  BFF completo. Un JotaBridge por sesión de voz.
        │
        ├──► [jota-orchestrator]  — Python/FastAPI — Puerto 8000
        │     HTTP POST /api/quick (NDJSON streaming)
        │     Headers: x-client-key, x-client-id
        │           │
        │           ├──► [jota-inference]  — C++/llama.cpp
        │           │     WebSocket (protocolo JSON propio)
        │           │
        │           └──► [jota-db]  — Python/FastAPI/SQLite — Puerto 8001
        │                 HTTP REST
        │
        ├──► [jota-transcriber]  — C++/whisper.cpp — Puerto 9000
        │     WebSocket (PCM Float32 in, JSON transcriptions out)
        │     Handshake: { type: "config", token: <client_key>, language, vad_thold }
        │
        └──► [jota-speaker]  — Python/FastAPI/Kokoro — Puerto 8005
              WebSocket (JSON tokens in, PCM16 audio out)
              Handshake: { type: "auth", token: <tts_token>, voice, speed }
```

---

## 2. Gateway — API pública

El gateway es el único punto de contacto con el exterior.

### WebSocket

| Endpoint | Descripción |
|---|---|
| `ws://gateway:8004/ws/stream` | Sesión bidireccional de voz (JotaBridge) |

### REST

| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/api/health` | Estado de todos los microservicios (público) |
| `GET` | `/api/models` | Modelos disponibles |
| `GET` | `/api/config` | Configuración del cliente |
| `PUT` | `/api/config` | Actualizar configuración (patch parcial) |
| `POST` | `/api/config/reset` | Restaurar configuración a defaults |
| `GET` | `/api/conversations` | Historial de conversaciones |
| `GET` | `/api/conversations/{id}/messages` | Mensajes de una conversación |
| `DELETE` | `/api/conversations/{id}` | Archivar conversación |

Todos los endpoints REST (excepto `/health`) requieren `X-API-Key` en el header, que se resuelve contra jota-db.

---

## 3. Flujo de Identidad

```
Cliente conecta → ws://gateway:8004/ws/stream
  Handshake: { "client_key": "abc123", ... }

  Gateway → GET /auth/session en jota-db
    { client: { id: "uuid-real", ... }, config: { stt_language: "es", tts_voice: "af_heart", ... } }

  Gateway → POST /api/quick en orchestrator
    headers: x-client-key: "abc123", x-client-id: "uuid-real"
    body: { text: "...", model_id, system_prompt_extra }

  Gateway → Transcriber WS
    Handshake: { type: "config", token: "abc123", language, vad_thold }

  Gateway → Speaker WS
    Handshake: { type: "auth", token: <TTS_TOKEN>, voice, speed }
```

Los valores de `language`, `vad_thold`, `voice`, `speed`, `model_id` y `system_prompt_extra` se leen de `ClientConfig` — no son globales.

---

## 4. ClientConfig

Almacenada en jota-db. Se carga en el handshake de voz y en cada request REST.

| Campo | Default | Usado en |
|---|---|---|
| `stt_language` | `"es"` | Handshake del transcriber |
| `stt_vad_thold` | `0.0` | Handshake del transcriber |
| `tts_voice` | `"af_heart"` | Handshake del speaker |
| `tts_speed` | `1.0` | Handshake del speaker |
| `preferred_model_id` | `null` | Payload al orchestrator |
| `system_prompt_extra` | `null` | Payload al orchestrator |
| `barge_in_enabled` | `true` | Bridge — cancelación de turno |
| `barge_in_min_chars` | `5` | Bridge — umbral de interrupción |
| `conversation_memory_limit` | `20` | Orchestrator — ventana de memoria |

Endpoints en jota-db: `GET/PUT /config/me`, `POST /config/me/reset`.

---

## 5. Protocolos de los microservicios

### jota-transcriber

**Handshake (gateway → transcriber):**
```json
{ "type": "config", "language": "es", "token": "<client_key>", "vad_thold": 0.0 }
```

**Respuesta `ready`:**
```json
{ "type": "ready", "protocol_version": 1, "session_id": "session-...", "config": { ... } }
```

**Mensajes del servidor:**
| Tipo | Cuándo |
|---|---|
| `ready` | Tras config válida |
| `transcription` | Parciales + final (`is_final: true`) |
| `warning` | Buffer ≥ 20 s (`code: "buffer_full"`) |
| `error` | Auth fallida, config inválida, etc. |

Fin de sesión: `{"type": "end"}` — dispara transcripción final y cierra.

**Auth del transcriber:**
- Dev: `AUTH_TOKEN=<token>`
- Prod: `AUTH_API_URL=http://jota-db:8001/auth` + `AUTH_API_SECRET`. Llama a `GET /auth/client`, caché de 300 s.

### jota-speaker

Auth vía `POST /auth/validate` en jota-db con `TTS_TOKEN`.

### jota-orchestrator

Acepta `POST /api/quick` con NDJSON streaming. Lee `x-client-key` y `x-client-id` del header para identificar al cliente y crear conversaciones con FK válida en jota-db.

---

## 6. Caché en el gateway

`src/core/cache.py` — `make_cache(maxsize, ttl)` → `(TTLCache, asyncio.Lock)`

| Recurso | TTL | Maxsize |
|---|---|---|
| `get_session()` (auth REST) | 60 s | 500 |
| `get_models()` | 300 s | 1 |

---

## 7. Issues abiertas

| Repo | Issue | Descripción | Prioridad |
|---|---|---|---|
| `jota-gateway` | #8 | Crear suite de tests y CI | P2 |
| `jota-orchestrator` | #15 | Eliminar `GET /models` (proxy redundante) | P3 |
| `jota-db` | #12 | Revisar/cerrar — ClientConfig ya implementada | P3 |
| `jota-transcriber` | #27 | Race condition `flushLoop`/`handleEnd` → duplicados `is_final` | P2 |

---

## 8. Estructura de ficheros clave

### jota-gateway

| Componente | Archivo |
|---|---|
| WebSocket BFF + JotaBridge | `src/services/bridge.py` |
| WS endpoint + handshake | `src/api/routes.py` |
| Auth dependency REST | `src/api/deps.py` |
| REST: config | `src/api/config_routes.py` |
| REST: conversaciones | `src/api/conversation_routes.py` |
| REST: modelos | `src/api/models_routes.py` |
| REST: health | `src/api/health_routes.py` |
| Cliente jota-db | `src/services/db_client.py` |
| Cliente orchestrator | `src/services/orchestrator_client.py` |
| Cliente transcriber | `src/services/transcriber_client.py` |
| Cliente TTS | `src/services/tts_client.py` |
| Caché TTL | `src/core/cache.py` |

### jota-orchestrator

| Componente | Archivo |
|---|---|
| Endpoint `/api/quick` (voz) | `src/api/quick.py` |
| Endpoint WebSocket chat | `src/api/chat/websocket.py` |
| Endpoint REST gestión | `src/api/rest.py` |
| Tool calling | `src/core/controller/input.py` |

### jota-db

| Componente | Archivo |
|---|---|
| Todos los modelos | `src/core/models.py` |
| Auth endpoints | `src/api/routers/auth.py` |
| Config endpoints (`/config/me`) | `src/api/routers/config.py` |
| Chat (conversaciones, mensajes) | `src/api/routers/chat.py` |
| Seguridad Bearer | `src/api/security.py` |

### jota-transcriber

| Componente | Archivo |
|---|---|
| Sesión WebSocket (protocolo) | `src/server/StreamingSession.h` |
| Whisper engine | `src/whisper/StreamingWhisperEngine.cpp` |
| Auth manager | `src/server/AuthManager.cpp` |
| Race condition (issue #27) | `src/server/StreamingSession.h:380-416, 471-556` |

---

*Documento vivo — actualizar al cerrar issues o cambiar contratos de API.*
