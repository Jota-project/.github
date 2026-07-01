# Jota — System Architecture

> Última actualización: 2026-07-01

---

## 1. Visión General

Jota es un asistente de voz distribuido. Un cliente físico (ESP32, app, web, Home Assistant, Termux) habla con un único punto de entrada (gateway), que orquesta varios microservicios especializados.

### Mapa de servicios

```
Physical Client (ESP32 / Web / App / Termux / Home Assistant)
        │  WebSocket  ws://gateway:8004/ws/stream
        │  REST HTTP  http://gateway:8004/api/*
        │  Handshake: { client_key, input_mode, output_mode, ... }
        ▼
  [jota-gateway]  — Python/FastAPI — Puerto 8004
  BFF completo. SQLite local para identidad y config.
        │
        ├──► [OpenClaw]  — orquestador externo open source
        │     WebSocket (puerto configurable; OpenClaw-managed)
        │     Token auth (OPENCLAW_TOKEN)
        │
        ├──► [jota-transcriber]  — C++17/whisper.cpp — Puerto 9000
        │     WebSocket (PCM Float32 in, JSON transcriptions out)
        │     Auth: AUTH_TOKEN estático o AUTH_API_URL externa
        │
        └──► [jota-speaker]  — Python/Kokoro — Puerto 8005
              WebSocket (JSON tokens in, PCM16 audio out)
              Wyoming TCP 20424 (Home Assistant native TTS)
              Auth: TTS_TOKEN estático
```

### Servicios Alternative (mantenidos por compatibilidad, menos desarrollo)

- `jota-orchestrator` — Orquestador LLM propio (Python/FastAPI, puerto 8000). Usar en su lugar: OpenClaw.
- `jota-inference` — Motor llama.cpp propio (C++, puerto 3000). Usar en su lugar: `llama.cpp` standalone con OpenClaw.

### Servicios Deprecated

- `jota-db` — Identidad/auth centralizado. El gateway ya tiene SQLite local. Mantenido solo como opción de auth externa para transcriber/speaker en setups legacy.

---

## 2. Gateway — API pública

El gateway es el único punto de contacto con el exterior. Implementa cuatro superficies:

### WebSocket

| Endpoint | Auth | Descripción |
|---|---|---|
| `ws://gateway:8004/ws/stream` | `client_key` en handshake JSON | Sesión interactiva de voz/texto |

### Health

| Endpoint | Auth | Descripción |
|---|---|---|
| `GET /healthz` | ninguna | Liveness |
| `GET /ready` | ninguna | Readiness (200 ok/degraded, 503 si OpenClaw no responde) |

### Admin (X-Admin-Token)

| Endpoint | Método | Descripción |
|---|---|---|
| `/admin/sessions` | GET | Sesiones activas y recientes |
| `/admin/sessions/{id}` | GET | Detalle con eventos y latencias |
| `/admin/orchestrators/{name}/status` | GET | Estado del orquestador (CONNECTED/RECONNECTING/DEGRADED) |
| `/admin/orchestrators/{name}/reconnect` | POST | Fuerza reconexión |
| `/admin/clients` | GET/POST | Listar/crear clientes (SQLite local) |
| `/admin/clients/{id}` | GET/PATCH/DELETE | Detalle/actualizar/borrar |
| `/admin/clients/{id}/rotate-key` | POST | Rotar `client_key` |

### OpenAI-compatible (LAN-only vía nginx)

| Endpoint | Método | Descripción |
|---|---|---|
| `/v1/models` | GET | Lista estática |
| `/v1/chat/completions` | POST | Chat completion; soporta `stream: true/false` |

Ver [`jota-gateway/README.md`](https://github.com/Jota-project/jota-gateway) para el detalle completo.

---

## 3. Flujo de Identidad

```
Cliente conecta → ws://gateway:8004/ws/stream
  Handshake: { "client_key": "abc123", ... }

  Gateway → SQLite local (data/gateway.db)
    SELECT * FROM clients WHERE client_key = 'abc123' AND is_active = 1
    → { id: "uuid-real", name: "...", stt_language: "es", tts_voice: "ef_dora", ... }

  Gateway → OpenClaw WebSocket
    Handshake con client_key + client_id

  Gateway → Transcriber WS
    Handshake: { type: "config", token: "abc123", language, vad_thold }

  Gateway → Speaker WS
    Handshake: { type: "auth", token: <TTS_TOKEN>, voice, speed }
```

Los valores de `language`, `vad_thold`, `voice`, `speed`, etc. se leen de `ClientRecord` (SQLite local) — no son globales.

---

## 4. ClientConfig / ClientRecord

Almacenada en SQLite local del gateway (`data/gateway.db`). Se carga en el handshake de voz y en cada request autenticado.

| Campo | Default | Usado en |
|---|---|---|
| `stt_language` | `"es"` | Handshake del transcriber |
| `stt_vad_thold` | `0.0` | Handshake del transcriber |
| `tts_voice` | `"af_heart"` | Handshake del speaker (ver §5.3 sobre Wyoming) |
| `tts_speed` | `1.0` | Handshake del speaker |
| `default_agent` | `null` | Override OpenClaw agent |
| `allowed_agents` | `null` | JSON list — agentes permitidos |
| `barge_in_enabled` | `true` | Bridge — cancelación de turno |
| `barge_in_min_chars` | `5` | Bridge — umbral de interrupción |
| `silence_timeout_s` | `2.0` | Bridge — silencio antes de evento |
| `max_silence_turns` | `3` | Bridge — turnos de silencio antes de cerrar |
| `push_enabled` | `true` | Si se aceptan push turns iniciados por el agente |
| `system_prompt_extra` | `null` | Appended al system prompt |

---

## 5. Protocolos de los microservicios

### 5.1 jota-transcriber

**Handshake (gateway → transcriber):**
```json
{ "type": "config", "language": "es", "token": "<client_key>", "vad_thold": 0.0 }
```

**Mensajes del servidor:** `ready`, `transcription` (con `is_final`), `warning` (code `buffer_full`), `error`. Fin de sesión: `{"type":"end"}`.

**Auth del transcriber:**
- Default: `AUTH_TOKEN=<token>` estático.
- Opcional: `AUTH_API_URL=<external URL>` + `AUTH_API_SECRET` para validar contra un backend externo. **Nota:** la opción recomendada hoy es `AUTH_TOKEN` estático; ver [issue abierta](https://github.com/Jota-project/jota-gateway/issues) sobre deprecar el uso de `jota-db` como auth backend.

### 5.2 jota-speaker

**WebSocket** (`/ws`):
```json
{ "type": "auth", "token": "<TTS_TOKEN>", "voice": "ef_dora", "speed": 1.0 }
```

Audio binario PCM16 24 kHz sale como frames binarios con header `[0xA1][turn_seq uint16 BE]`.

**Wyoming TCP** (puerto 20424, opcional, `JOTA_WYOMING_ENABLED=true`):
Permite usar `jota-speaker` como TTS nativo de Home Assistant.

### 5.3 OpenClaw

Orquestador LLM externo. El gateway mantiene un `OpenClawClient` singleton envuelto en `ReconnectingOpenClawClient` (estados CONNECTED / RECONNECTING / DEGRADED).

Configuración gateway:
- `OPENCLAW_HOST`
- `OPENCLAW_PORT`
- `OPENCLAW_TOKEN`

Ver docs upstream de OpenClaw para el protocolo detallado.

### 5.4 jota-orchestrator (Alternative)

Aceptaba `POST /api/quick` con NDJSON streaming. Lee `x-client-key` y `x-client-id` del header. **Mantenido por compatibilidad; el camino recomendado es OpenClaw (§5.3).**

---

## 6. Caché en el gateway

`src/core/cache.py` — `make_cache(maxsize, ttl)` → `(TTLCache, asyncio.Lock)`

| Recurso | TTL | Maxsize |
|---|---|---|
| `get_verified_client()` (auth REST) | 60 s | 500 |
| `get_models()` | 300 s | 1 |

---

## 7. Clientes

### jota-voice

Cliente Termux/Android que reemplaza `wyoming-satellite` + HA voice pipeline. Conecta vía WS streaming directo al gateway, reduciendo latencia de ~5-6s a <2s.

Arquitectura local en el dispositivo:
- `wyoming-openwakeword` (puerto 10401) → wake word detection
- `jota-voice-client` → WS streaming hacia gateway
- `jota-display` (puerto 8766) → UI kiosk

Ver [jota-voice/docs/spec.md](https://github.com/Jota-project/jota-voice/blob/main/docs/spec.md) para detalle.

### jota-display

UI kiosk (Vue 3 + SSE) que muestra estado de la conversación y permite controlar Home Assistant. Diseñada para correr en el mismo dispositivo Android que `jota-voice` (o en una Raspberry Pi / tablet separada).

Ver [jota-display/docs/SPEC.md](https://github.com/Jota-project/jota-display/blob/main/docs/SPEC.md) para la hoja de ruta completa.

---

## 8. Issues activas

| Repo | Issue | Descripción | Prioridad |
|---|---|---|---|
| `jota-gateway` | #52 | `/v1/*` endpoints expuestos externamente sin auth — conflicto de compat con HA | P1 |
| `jota-gateway` | #50 | E2E test suite con Docker Compose | P2 |
| `jota-gateway` | #49 | `DELETE /api/conversations/{id}` — endpoint de archive | P3 |
| `jota-gateway` | #48 | Cachear `get_verified_client()` para evitar round-trip por request | P2 |
| `jota-gateway` | (nueva) | Deprecar jota-db como auth backend en transcriber/speaker | P3 |
| `jota-transcriber` | #27 | Race condition `flushLoop`/`handleEnd` → duplicados `is_final` | P2 |

---

## 9. Estructura de ficheros clave

### jota-gateway

| Componente | Archivo |
|---|---|
| WebSocket BFF + JotaBridge | `src/services/bridge.py` |
| WS endpoint + handshake | `src/api/routes.py` |
| OpenClaw client (singleton + reconexión) | `src/services/openclaw_client.py` |
| DB local (SQLModel) | `src/db/models.py`, `src/db/database.py` |
| Auth dependency REST | `src/api/deps.py` |
| Caché TTL | `src/core/cache.py` |

### jota-orchestrator (Alternative)

| Componente | Archivo |
|---|---|
| Endpoint `/api/quick` | `src/api/quick.py` |
| Tool calling | `src/core/controller/input.py` |

### jota-transcriber

| Componente | Archivo |
|---|---|
| Sesión WebSocket | `src/server/StreamingSession.h` |
| Whisper engine | `src/whisper/StreamingWhisperEngine.cpp` |
| Auth manager | `src/server/AuthManager.cpp` |

### jota-speaker

| Componente | Archivo |
|---|---|
| WebSocket TTS | `src/main.py` (FastAPI app) |
| Wyoming server | `src/wyoming/` |
| Kokoro engine wrapper | `src/tts/kokoro_engine.py` |

### jota-db (Deprecated)

| Componente | Archivo |
|---|---|
| Routers `/internal/` y `/admin/` | `src/api/routers/admin/` |
| Modelos | `src/core/models.py` |
| Bearer security | `src/api/security.py` |

---

## 10. Legacy / Deprecated

### Por qué `jota-orchestrator` y `jota-inference` están en modo Alternative

Estos dos servicios formaban el camino LLM original de Jota. Recientemente hemos adoptado **OpenClaw**, un orquestador open source más completo y con un ecosistema de plugins más amplio. La combinación OpenClaw + `llama.cpp` cubre los mismos casos de uso (LLM local, tools, memoria) con menos código propio que mantener.

`jota-orchestrator` y `jota-inference` siguen siendo **útiles** para setups que necesitan control total sobre la inferencia y no quieren depender de un proyecto externo. Ambos están congelados en su estado funcional actual y reciben parches solo para issues críticas.

**Recomendación:** nuevos deployments deberían usar la combinación OpenClaw + `llama.cpp` (rutas Maintained). Para setups existentes, no es necesario migrar inmediatamente — el gateway sigue siendo compatible con `jota-orchestrator` vía `x-client-key`/`x-client-id`.

### Por qué `jota-db` está Deprecated

A partir de `jota-gateway` v1.9.0, el gateway mantiene su propia base SQLite local para identidad, configuración de clientes y admin API. La dependencia de `jota-db` para esas funciones es historia.

`jota-db` se mantiene por dos razones:
1. **Auth externa opcional.** `jota-transcriber` y `jota-speaker` aún pueden usarlo como backend de validación de tokens (`AUTH_API_URL`).
2. **Setups centralizados.** Algunos deployments prefieren un único servicio de auth para todos los microservicios.

La dirección es eliminar la dependencia por defecto en transcriber/speaker (issue abierta en `jota-gateway`).

---

*Documento vivo — actualizar al cerrar issues o cambiar contratos de API.*
