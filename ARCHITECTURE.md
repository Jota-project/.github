# Jota — Arquitectura del Sistema

> Última actualización: 2026-04-05

---

## 1. Visión General

Jota es un asistente de voz distribuido. Un cliente físico (ESP32, app, web) habla con un único punto de entrada (gateway), que orquesta varios microservicios especializados.

> **Sistema single-user, multi-device.** Jota está diseñado para un único propietario con múltiples dispositivos cliente (altavoz del salón, cocina, habitación, jardín, etc.). Los "clientes" son dispositivos, no usuarios independientes. Toda la identidad y configuración pertenece al mismo dueño.

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

## 7. ServiceConfig — Configuración centralizada de servicios

Cada microservicio tiene parámetros de operación propios que hoy viven en variables de entorno. El objetivo es centralizar esta configuración en jota-db para permitir modificación en caliente y administración unificada.

### Tabla `service_config`

| Campo | Tipo | Descripción |
|---|---|---|
| `service` | TEXT | Servicio al que pertenece (`transcriber`, `orchestrator`, `speaker`, `gateway`, `inference`) |
| `key` | TEXT | Nombre del parámetro, ej. `model`, `audio.chunk_ms` |
| `value` | JSON | Valor de cualquier tipo |
| `description` | TEXT | Descripción legible |

`PRIMARY KEY (service, key)`

**Seeding desde `.env`:** al arrancar jota-db, si la tabla está vacía se puebla desde variables de entorno. El `.env` sigue siendo la fuente de bootstrap; la DB actúa como override persistente una vez estable.

### Endpoints admin (jota-db)

| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/admin/config` | Listar toda la configuración |
| `GET` | `/admin/config/{service}` | Config de un servicio |
| `PUT` | `/admin/config/{service}/{key}` | Crear o actualizar valor |
| `POST` | `/admin/config/{service}/reset` | Restaurar defaults desde `.env` |

Expuestos en gateway como `/api/admin/config/*`.

---

## 8. InferenceProvider — Soporte multi-provider de IA

El sistema soporta múltiples backends de inferencia: el motor local (jota-inference/llama.cpp), APIs externas (OpenAI, Anthropic) y servidores custom (VPS del usuario). Los providers disponibles son configuración del sistema (del propietario), no por dispositivo.

### Tabla `inference_provider`

| Campo | Tipo | Descripción |
|---|---|---|
| `id` | UUID | Identificador |
| `name` | TEXT | Nombre display, ej. "OpenAI GPT-4o" |
| `type` | TEXT | `local` \| `openai` \| `anthropic` \| `custom` |
| `base_url` | TEXT? | URL base (requerida para `local` y `custom`) |
| `api_key` | TEXT? | API key (para `openai` y `anthropic`) |
| `default_model_id` | TEXT? | Modelo por defecto del provider |
| `is_active` | BOOL | Si está disponible para uso |
| `extra_config` | JSON? | Config adicional específica del provider |

### Selección de provider

El orchestrator lee `orchestrator.default_provider_id` de ServiceConfig. La selección por request sigue este orden:
1. `ClientConfig.preferred_model_id` — override por dispositivo
2. `orchestrator.default_provider_id` — default del sistema

### Capa de abstracción en jota-orchestrator

```
LLMProvider (interfaz)
├── LocalProvider       → WebSocket a jota-inference (comportamiento actual)
├── OpenAIProvider      → openai Python SDK
├── AnthropicProvider   → anthropic Python SDK
└── CustomHTTPProvider  → endpoint compatible OpenAI en VPS propio
```

jota-inference permanece sin cambios como backend local.

### Endpoints admin (jota-db)

| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/admin/providers` | Listar providers |
| `POST` | `/admin/providers` | Crear provider |
| `PUT` | `/admin/providers/{id}` | Actualizar (API key, activar/desactivar) |
| `DELETE` | `/admin/providers/{id}` | Desactivar |

Expuestos en gateway como `/api/admin/providers/*`.

> Spec completa: `docs/superpowers/specs/2026-04-08-serviceconfig-providers-design.md`

---

## 9. Issues abiertas

| Repo | Issue | Descripción | Prioridad |
|---|---|---|---|
| `jota-db` | PR #15 | ServiceConfig + InferenceProvider + AdminUser + /internal/ + /admin/ | P1 — en revisión |
| `jota-gateway` | #37 | Proxy `/api/admin/*` a jota-db | P1 |
| `jota-inference` | #23 | API HTTP compatible OpenAI (`/v1/chat/completions`) | P1 |
| `jota-orchestrator` | #21 | Capa `LLMProvider` + lectura desde `/internal/` | P1 |
| `jota-transcriber` | #29 | InternalService + leer ServiceConfig desde `/internal/` | P2 |
| `jota-speaker` | #1 | InternalService + leer ServiceConfig desde `/internal/` | P2 |
| `jota-transcriber` | #27 | Race condition `flushLoop`/`handleEnd` → duplicados `is_final` | P2 |
| `jota-orchestrator` | #15 | Eliminar `GET /models` (proxy redundante) | P3 |
| `jota-db` | #12 | Revisar/cerrar — ClientConfig ya implementada | P3 |

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
