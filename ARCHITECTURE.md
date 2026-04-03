# Jota — Análisis Arquitectónico y Plan de Mejora

> Documento generado el 2026-03-23. Última actualización: 2026-04-02.

---

## 1. Visión General

Jota es un asistente de voz distribuido ("Alexa con cerebro LLM"). Un cliente físico (ESP32, app, web) habla con un único punto de entrada (gateway), que orquesta varios microservicios especializados.

### Mapa de servicios

```
Physical Client (ESP32 / Web / App)
        │  WebSocket ws://gateway:8004/ws/stream
        │  REST     http://gateway:8004/api/...
        │  Handshake: { client_key, input_mode, output_mode, ... }
        │  (PCM Float32 mic audio + JSON text)
        ▼
  [jota-gateway]  — Python/FastAPI — Puerto 8004
  BFF completo. Un JotaBridge por conexión WS. DbClient compartido.
        │
        ├──► [jota-db]  — Python/FastAPI/SQLite — Puerto 8001   ← SERVICIO DE PRIMER NIVEL
        │     HTTP REST (DbClient)
        │     · Handshake: GET /auth/session → Client + ClientConfig
        │     · Config:    GET/PUT/POST /config/me
        │     · Chat:      GET /conversations, GET /conversations/{id}/messages
        │
        ├──► [jota-orchestrator]  — Python/FastAPI — Puerto 8000
        │     HTTP POST /api/quick (NDJSON streaming)
        │     Header: x-client-key: <client_key>, x-client-id: <uuid>
        │           │
        │           ├──► [jota-inference]  — C++/llama.cpp
        │           │     WebSocket (protocolo JSON propio)
        │           │
        │           └──► [jota-db]  :8001  (conversaciones, mensajes, modelos)
        │
        ├──► [jota-transcriber]  — C++/whisper.cpp — Puerto 9000
        │     WebSocket (PCM Float32 in, JSON transcriptions out)
        │     Handshake: { type: "config", token: <client_key>, language, vad_thold }
        │
        └──► [jota-speaker]  — Python/FastAPI/Kokoro — Puerto 8005
              WebSocket (JSON tokens in, PCM16 audio out)
```

---

## 2. Estado Actual: Diagnóstico

### Lo que funciona

- Separación de responsabilidades clara por microservicio
- jota-db como fuente de verdad con versionado optimista
- Auth interna bien diseñada: `InferenceClient` (servicios) vs `Client` (externos)
- Pipeline de tool-calling maduro en orchestrator: detección dual, permisos por rol, re-inferencia
- Streaming end-to-end sin bloqueo (tokens LLM → cliente)
- REST API básica ya existente en el orchestrator (`rest.py`)
- `client_key` del cliente propagado a todos los servicios downstream ✅ (PR #17)
- Transcripciones parciales reenviadas al cliente en tiempo real ✅ (PR #16)
- Deduplicación de transcripciones finales antes del orchestrator ✅ (PR #7)
- Manejo de fallos del transcriber con notificación al cliente ✅ (PR #6)

### Problema central (pendiente)

> El gateway propaga la `client_key` pero aún no la resuelve a un UUID real.
> `JotaBridge.client_id` sigue siendo el string de la key, no el UUID de jota-db.
> Esto bloquea la correcta asociación de conversaciones en jota-db.

### Problemas específicos restantes

**Identidad parcialmente resuelta (Fase 1 en progreso)**
- `client_key` llega al gateway y se reenvía a cada servicio ✅
- El UUID real del `Client` en jota-db no se resuelve todavía durante el handshake ❌
- El orchestrator sigue recibiendo la key como `user_id` en lugar del UUID → FK violation en jota-db

**Configuración inexistente por cliente**
- No hay mecanismo para preferencias por dispositivo (voz TTS, idioma STT, modelo LLM)
- `language` hardcodeado a `"es"` (issue #14), `vad_thold` a `0.0` (issue #15)
- Pendiente Fase 2: ClientConfig en jota-db

**No hay API REST en el gateway**
- El cliente solo puede interactuar por WebSocket
- Para consultar modelos, historial, config → necesita conectar directamente al orchestrator
- Pendiente Fase 3

---

## 3. Protocolos de Comunicación por Servicio

| Servicio | Entrada | Protocolo | Salida |
|---|---|---|---|
| gateway ← cliente (WS) | WS `ws://gateway:8004/ws/stream` | Binary PCM Float32 + JSON | Binary PCM16 + JSON |
| gateway ← cliente (REST) | HTTP `http://gateway:8004/api/...` | JSON | JSON |
| gateway → jota-db | HTTP REST (`DbClient`) | JSON | JSON |
| gateway → orchestrator | HTTP POST `/api/quick` | NDJSON streaming | NDJSON tokens |
| gateway → transcriber | WS `ws://transcriber:9000` | Binary PCM Float32 + JSON config | JSON transcriptions |
| gateway → speaker | WS `ws://speaker:8005/ws` | JSON tokens | Binary PCM16 |
| orchestrator → inference | WS persistente | JSON protocolo propio | JSON tokens streaming |
| orchestrator → jota-db | HTTP REST | JSON | JSON |
| inference → jota-db | HTTP REST | JSON | JSON (auth en startup) |
| speaker → jota-db | HTTP REST `POST /auth/validate` | JSON | JSON (auth por sesión) |

### 3.1 Protocolo del transcriber (auditado en detalle)

**Handshake (cliente → servidor):**
```json
{ "type": "config", "language": "es", "token": "<client_key>", "vad_thold": 0.0 }
```

**Respuesta ready (servidor → cliente):**
```json
{ "type": "ready", "protocol_version": 1, "session_id": "session-...", "config": { "language": "es", "sample_rate": 16000, "beam_size": 5 } }
```

**Audio:** PCM Float32 LE, 16kHz, mono, frames de 4–1.048.576 bytes (múltiplo de 4). Rate limit: 600 KB/3 s.

**Mensajes del servidor:**
| Tipo | Cuándo | Campos clave |
|---|---|---|
| `ready` | Tras config válida | `session_id`, `protocol_version`, `config` |
| `transcription` | Continuo (parciales) + al recibir `end` (final) | `text`, `is_final` |
| `warning` | Buffer ≥ 20 s | `code: "buffer_full"`, `message` |
| `error` | Auth fallida, config inválida, etc. | `code`, `message` |

**Códigos de error:** `AUTH_REQUIRED`, `AUTH_FAILED`, `NOT_CONFIGURED`, `INVALID_MESSAGE`, `UNKNOWN_TYPE`, `PARSE_ERROR`, `AUDIO_ERROR`, `CONFIG_ERROR`.

**Mensaje `end`:** `{"type": "end"}` — dispara transcripción final y cierra la conexión.

**Guardia de alucinaciones:** el servidor rechaza internamente texto con >500 chars, 4+ palabras idénticas consecutivas, o 4+ bigramas repetidos.

**Ciclo de transcripción:** `flushLoop` corre cada 200 ms. Emite parcial si hay ≥250 ms de audio nuevo o ≥400 ms de silencio. Requiere buffer mínimo de 2 s antes de la primera inferencia.

### 3.2 Autenticación del transcriber

El transcriber puede funcionar en dos modos configurados por env vars del servidor:

```
Modo estático (dev):    AUTH_TOKEN=<token>
Modo API (producción):  AUTH_API_URL=http://jota-db:8001/auth
                        AUTH_API_SECRET=<API_SECRET_KEY de jota-db>
```

En modo API, valida el `token` del handshake llamando a:
```
GET /auth/client
Authorization: Bearer <AUTH_API_SECRET>
X-API-Key: <token recibido del cliente>
```
Espera `{ "is_active": true }`. Caché de 300 s por token. Fail-closed si jota-db no responde.

---

## 4. Flujo de Identidad — Estado Actual

```
ESP32 conecta → ws://gateway:8004/ws/stream
  Handshake: { "client_key": "abc123", "input_mode": "audio", ... }

  Gateway → POST /api/quick
    header: x-client-key: "abc123"     ← key del cliente, NO shared key
    body: { text: "...", user_id: "abc123" }  ← ⚠️ sigue siendo la key, no el UUID

  Orchestrator valida x-client-key → obtiene Client de jota-db
    Crea conversación con:
      client_id = "abc123"  ← ⚠️ debería ser Client.id (UUID), no la key
      ↑ FK violation en jota-db si la key no coincide con ningún id

  Gateway → Transcriber WS
    Handshake: { "token": "abc123", ... }
    Transcriber (modo API) → GET /auth/client (X-API-Key: abc123) → jota-db
    ✅ Auth correcta — el transcriber resuelve la identidad por su cuenta
```

### Lo que falta para completar Fase 1

El gateway necesita llamar él mismo a jota-db durante el handshake para:
1. Validar que la key existe y el cliente está activo (rechazar la sesión si no)
2. Obtener el UUID real para usarlo como `client_id` interno y como `user_id` al orchestrator

```
Flujo objetivo:
  Gateway recibe handshake con client_key
    → GET /auth/client (jota-db) → Client { id: "uuid-real", ... }
    → JotaBridge.client_id = "uuid-real"
    → orchestrator recibe user_id: "uuid-real"  ← FK válida en jota-db ✅
    → transcriber recibe token: "abc123"        ← valida por su cuenta ✅
```

---

## 5. Issues — Estado Actualizado

### Bloquean el funcionamiento completo del sistema

| Prioridad | Issue | Síntoma | Estado |
|---|---|---|---|
| P0 | `jota-db#1` | `/api/quick` rechaza a todos los clientes (`client_type` ausente) | ✅ Cerrada |
| P0 | `jota-db#2` | Tool calling en loop (`role=tool` rechazado con 422) | ✅ Cerrada |
| P0 | `jota-orchestrator#9` | Conversaciones con FK inválida en jota-db | ⚠️ Pendiente |
| P1 | `jota-db#3` | Modelo nunca se asocia a conversaciones | ✅ Cerrada |
| P1 | `jota-db#4` | Auth del TTS rota | ✅ Cerrada |
| P1 | `jota-inference#6` | Tool calling roto (relacionado con db#2) | ⚠️ Pendiente |

### Degradan la experiencia — jota-gateway

| Prioridad | Issue | Estado |
|---|---|---|
| P2 | #3 Múltiples sesiones de inferencia por prompt | ✅ PR #7 |
| P2 | #4 Transcripción no en tiempo real | ✅ PR #16 |
| P3 | #5 Token hardcodeado | ✅ PR #17 |
| P3 | #2 Client ID / identidad (Fase 1) | ⚠️ Parcial — key propagada, UUID pendiente |
| P3 | #1 API REST (Fase 3) | 📋 Pendiente |

### Protocolo del transcriber — jota-gateway (nuevas)

| Prioridad | Issue | Descripción |
|---|---|---|
| P2 | #10 | `TranscriberMessage` sin campo `code` — se pierden códigos de error/warning |
| P2 | #11 | Warning `buffer_full` descartado silenciosamente |
| P2 | #14 | Idioma hardcodeado a `"es"` — no se lee del handshake |
| P3 | #9 | Campo `publish_mqtt` muerto en el handshake |
| P3 | #12 | `TranscriberConfig` schema definido pero no se usa |
| P3 | #13 | `session_id` del mensaje `ready` se pierde |
| P3 | #15 | `vad_thold` hardcodeado a `0.0` |

### jota-transcriber

| Prioridad | Issue | Descripción |
|---|---|---|
| P2 | #27 | Race condition entre `flushLoop` y `handleEnd` → posibles duplicados `is_final=true` |
| P4 | #22 | Alucinaciones eliminan el resultado; podría conservarse una copia |

---

## 6. Decisiones Arquitectónicas

### Gateway como BFF completo *(en curso)*

El gateway evoluciona de "proxy WebSocket" a **Backend-For-Frontend**:
- Único punto de contacto entre exterior e interior
- REST API para gestión + WebSocket para tiempo real
- Identidad del cliente resuelta en el gateway y propagada como contexto verificado

### jota-db como servicio de primer nivel del gateway *(nueva decisión)*

jota-db deja de ser solo un destino de otros servicios internos para convertirse en la **fuente de verdad de identidad y configuración** del gateway. El gateway mantiene un `DbClient` compartido con el que:

1. **En el handshake WS**: resuelve `client_key → Client + ClientConfig` en una sola llamada y construye el contexto completo de sesión
2. **En las rutas REST**: actúa de proxy/facade hacia jota-db para que el cliente externo pueda leer y modificar su propia configuración, historial, etc.

Los servicios internos (orchestrator, transcriber, speaker) siguen comunicándose con jota-db de forma independiente (zero-trust). El gateway **no verifica** esos servicios — ya se validan solos. Solo verifica al cliente externo.

**Nuevo endpoint propuesto en jota-db:**
```
GET /auth/session
Authorization: Bearer {JOTA_DB_API_KEY}
X-API-Key: {client_key}

→ { "client": { "id": "uuid", "is_active": true, ... },
    "config": { "stt_language": "es", "tts_voice": "af_heart", ... } }
```
Una sola llamada al handshake en lugar de dos. Si el cliente no tiene `ClientConfig`, jota-db lo auto-crea con defaults.

```
                    ┌─────────────────────────────────────────────┐
                    │              EXTERIOR                        │
                    │   ESP32, App, Web, CLI...                    │
                    └──────────────────┬──────────────────────────┘
                                       │  WS /ws/stream  (voz)
                                       │  REST /api/...  (gestión)
                                       │  Auth: X-API-Key: {client_key}
                                       │
                    ┌──────────────────▼──────────────────────────┐
                    │              jota-gateway :8004              │
                    │                                              │
                    │   DbClient ──────────────────────────────►  │
                    │   · handshake: GET /auth/session             │
                    │   · config:    GET/PUT/POST /config/me       │
                    │   · historial: GET /conversations/...        │
                    │                                              │
                    │   REST  /api/config                          │
                    │         /api/conversations                   │
                    │         /api/models                          │
                    │         /api/health                          │
                    │                                              │
                    │   WS    /ws/stream  (voz en tiempo real)     │
                    └──┬──────────┬──────────┬──────────┬─────────┘
                       │          │          │          │
              ┌────────▼──┐ ┌────▼────┐ ┌──▼────┐ ┌──▼──────┐
              │orchestrator│ │transcrib│ │speaker│ │ jota-db │
              │  :8000     │ │  :9000  │ │ :8005 │ │  :8001  │
              └────────────┘ └─────────┘ └───────┘ └─────────┘
                    │               │         │          ▲
                    └───────────────┴─────────┴──────────┘
                         (cada servicio valida por su cuenta)
```
---

## 7. Plan de Implementación

### Fase 0 — Bugs críticos ✅ COMPLETADA (salvo jota-orchestrator #9)

| Tarea | Estado |
|---|---|
| `jota-db`: añadir `client_type`, `role=tool`, `extra_data`, renombrar `model_id` | ✅ |
| `jota-db`: endpoint `/auth/validate` para jota-speaker | ✅ |
| `jota-gateway`: deduplicar transcripciones antes del orchestrator | ✅ PR #7 |
| `jota-gateway`: reenviar transcripciones parciales al cliente | ✅ PR #16 |
| `jota-gateway`: token hardcodeado → client_key del cliente | ✅ PR #17 |
| `jota-gateway`: manejo de fallos del transcriber (watchdog + dropped) | ✅ PR #6 |
| `jota-orchestrator`: usar UUID real como client_id | ⚠️ Issue #9 pendiente |

### Fase 1 — Identity + Config: jota-db como servicio de primer nivel (en progreso)

Fusiona el antiguo "Fase 1 (UUID)" con "Fase 2 (ClientConfig)". Son la misma iniciativa.

**Avanzado:**
- ✅ `client_key` en el schema `Handshake`
- ✅ Path `/ws/stream` sin `{client_id}`
- ✅ `client_key` propagada como auth a orchestrator y transcriber
- ✅ `ORCHESTRATOR_API_KEY` eliminado

**Paso 1 — Extender jota-db:**

1. Añadir modelo `ClientConfig` a `models.py` + migración:

   ```python
   class ClientConfig(BaseUUIDModel, table=True):
       client_id: str = Field(foreign_key="client.id", unique=True)
       stt_language: str = Field(default="es")
       stt_model: Optional[str] = None
       stt_vad_thold: float = Field(default=0.0)
       tts_voice: str = Field(default="af_heart")
       tts_speed: float = Field(default=1.0)
       preferred_model_id: Optional[str] = Field(default=None, foreign_key="aimodel.id")
       system_prompt_extra: Optional[str] = None
       barge_in_enabled: bool = Field(default=True)
       barge_in_min_chars: int = Field(default=5)
       conversation_memory_limit: int = Field(default=20)
   ```

2. Auto-crear `ClientConfig` con defaults al crear un `Client`

3. Crear router `src/api/routers/config.py` con:

   ```text
   GET  /config/me       → ClientConfig del cliente autenticado
   PUT  /config/me       → actualización parcial
   POST /config/me/reset → restaura a defaults
   ```

4. Nuevo endpoint de sesión en `auth.py`:

   ```text
   GET /auth/session     → { client: Client, config: ClientConfig }
   ```

   (auto-crea ClientConfig si no existe)

**Paso 2 — DbClient en el gateway:**

1. Añadir `JOTA_DB_BASE_URL` y `JOTA_DB_API_KEY` a `config.py`

2. Crear `src/services/db_client.py` — cliente HTTP hacia jota-db:

   ```python
   class DbClient:
       async def get_session(client_key: str) -> tuple[Client, ClientConfig]
       async def get_config(client_id: str) -> ClientConfig
       async def update_config(client_id: str, patch: dict) -> ClientConfig
       async def reset_config(client_id: str) -> ClientConfig
       async def get_conversations(client_id: str) -> list[Conversation]
       async def get_messages(conversation_id: str) -> list[Message]
   ```

3. Instanciar `DbClient` como singleton compartido en el gateway

4. En `routes.py`, tras handshake, llamar `DbClient.get_session(client_key)`:
   - Si 401/403 → cerrar WS `1008` (key inválida)
   - Si error de red → cerrar WS `1011` (servicio no disponible)

5. `JotaBridge` recibe `client: Client` + `config: ClientConfig` completos

6. Orchestrator recibe `user_id` = `client.id` (UUID) → FK válida en jota-db ✅

7. Resuelve issues #14 (idioma) y #15 (vad_thold) como efecto inmediato

### Fase 2 — Gateway REST API pública

Con `DbClient` ya disponible, el gateway expone sus propios endpoints REST al cliente externo. Auth: `X-API-Key: {client_key}` → `get_verified_client()` dependency.

**Endpoints en el gateway:**
```
# Configuración (proxied a jota-db via DbClient)
GET    /api/config                        → ClientConfig del cliente
PUT    /api/config                        → actualiza campos parcialmente
POST   /api/config/reset                  → restaura defaults

# Historial (proxied a jota-db via DbClient)
GET    /api/conversations                 → lista conversaciones
GET    /api/conversations/{id}/messages   → mensajes de una conversación
DELETE /api/conversations/{id}            → archiva conversación

# Modelos (proxied a orchestrator)
GET    /api/models                        → modelos disponibles

# Sistema
GET    /api/health                        → estado de todos los servicios internos
```

**Pasos:**

1. Crear dependency `get_verified_client(x_api_key: str)` → llama `DbClient.get_session()`
2. Crear `src/api/http_routes.py` con los endpoints anteriores
3. Montar el router en `main.py`

### Fase 3 — Propagación de ClientConfig a los servicios

Al iniciar sesión de voz, el gateway ya tiene `Client` + `ClientConfig` (desde Fase 1). Inyectar esos valores en cada conexión:

```python
# STT
transcriber.connect(language=config.stt_language, token=client.client_key, vad_thold=config.stt_vad_thold)

# TTS
tts.connect(token=settings.TTS_TOKEN, voice=config.tts_voice, speed=config.tts_speed)

# Orchestrator
orchestrator = OrchestratorClient(client_id=client.id)
# payload incluirá model_id=config.preferred_model_id, system_extra=config.system_prompt_extra
```

---

## 8. Resumen del Estado Objetivo por Fase

```
Fase 0 → Sistema funciona sin errores de schema ni de identidad en DB       ✅ (salvo orquestrador #9)
Fase 1 → Identidad real del cliente en todo el sistema (UUID, no string)    ⚠️ En progreso
Fase 2 → Cada cliente tiene su perfil de configuración en jota-db           📋 Pendiente
Fase 3 → El cliente puede leer y modificar todo desde el gateway (REST)     📋 Pendiente
Fase 4 → Los servicios (TTS, STT, LLM) se comportan según el perfil        📋 Pendiente
```

---

## 9. Código Clave — Referencias

### jota-gateway
| Componente | Archivo |
|---|---|
| WebSocket endpoint + handshake | `src/api/routes.py` |
| `JotaBridge` (core del proxy) | `src/services/bridge.py` |
| Schema `Handshake` (incluye `client_key`) | `src/models/schemas.py` |
| Config | `src/core/config.py` |
| Watchdog de silencio del transcriber | `src/services/bridge.py` → `_transcription_watchdog()` |
| Spec de manejo de fallos del transcriber | `docs/superpowers/specs/2026-03-14-transcriber-failure-handling-design.md` |

### jota-orchestrator
| Componente | Archivo |
|---|---|
| Endpoint `/api/quick` (voz) | `src/api/quick.py` |
| Endpoint WebSocket chat | `src/api/chat/websocket.py` |
| Endpoint REST gestión | `src/api/rest.py` |
| `client_id` incorrecto (issue #9) | `src/core/memory.py:138` |
| Tool calling loop | `src/core/controller/input.py` |

### jota-db
| Componente | Archivo |
|---|---|
| Todos los modelos | `src/core/models.py` |
| Auth endpoints (incluye `GET /auth/client`) | `src/api/routers/auth.py` |
| Validación de Bearer token servicio-a-servicio | `src/api/security.py` |
| Chat (conversaciones, mensajes) | `src/api/routers/chat.py` |

### jota-transcriber
| Componente | Archivo |
|---|---|
| Sesión WebSocket (protocolo completo) | `src/server/StreamingSession.h` |
| Whisper engine (sliding window, buffer) | `src/whisper/StreamingWhisperEngine.cpp` |
| Auth manager (estático vs API) | `src/server/AuthManager.cpp` |
| Cliente HTTP hacia jota-db | `src/auth/ApiAuthClient.cpp` |
| Caché de tokens | `src/auth/AuthCache.h` |
| Guardia de alucinaciones | `src/server/HallucinationGuard.h` |
| Race condition `flushLoop`/`handleEnd` (issue #27) | `src/server/StreamingSession.h:380-416, 471-556` |

### jota-speaker
| Componente | Archivo |
|---|---|
| Auth endpoint erróneo | `src/core/config.py:10` |
| Auth provider jota-db | `src/auth/jota_db.py` |

---

*Documento vivo — actualizar conforme se completan fases.*
