# Jota — Tareas pendientes de implementación

> Actualizado el 2026-04-03. Ver análisis completo en `jota-architecture.md`.

---

## Estado actual

### jota-db — Fase 0 COMPLETADA

Todas las issues de Fase 0 están cerradas y mergeadas en `main`.

| Issue | Título | Estado |
|---|---|---|
| #1 | client_type en Client | ✅ PR #5 mergeado |
| #2 | role=tool en MessageRole + metadata en Message | ✅ PR #5 (campo llamado `extra_data`) |
| #3 | Rename ai_model_id → model_id en Conversation | ✅ PR #5 mergeado |
| #4 | /auth/validate | ✅ Cerrada — error de diseño en jota-speaker |
| #6 | HOST_MODELS_DIR fallback hardcodeado | ✅ PR #9 (pendiente merge) |

> ⚠️ **Cambio de API**: el campo `metadata` en `Message` y `MessageCreate` fue renombrado a `extra_data`.
> jota-orchestrator debe actualizar sus llamadas a `POST /chat/{id}/messages`.

---

## Fase 0 pendiente — otros repos

### jota-orchestrator — Issue #9

El orchestrator usa el string libre del path del gateway como `client_id` en lugar del UUID real de jota-db.

- [ ] En `src/core/memory.py` función `create_conversation()`: cambiar `"client_id": user_id` por `"client_id": client_id` donde `client_id` viene de `client_data["id"]` (el UUID real de jota-db)
- [ ] Verificar que en `src/api/quick.py` y `src/api/chat/websocket.py` se pasa `client_data["id"]` correctamente a todas las llamadas de `memory_manager`
- [ ] Verificar `client_type` check en `src/api/quick.py:183` — funciona ahora que jota-db tiene el campo
- [ ] Actualizar llamadas a `POST /chat/{id}/messages`: usar `extra_data` en lugar de `metadata`

**Archivos a tocar:** `src/core/memory.py`, `src/api/quick.py`, `src/api/chat/websocket.py`

---

### jota-gateway — Issues cerradas en esta sesión

| Issue | Título | PR |
|---|---|---|
| #3 | Varias sesiones de inferencia por prompt | ✅ PR #7 mergeado |
| #4 | Transcripción no en directo | ✅ PR #16 mergeado |
| #5 | Token hardcodeado del transcriber | ✅ PR #17 mergeado (ver nota) |

> **Nota #5**: La solución no fue mover el token a config, sino eliminarlo. Ahora el cliente envía
> su `client_key` en el handshake y el gateway la reenvía directamente al transcriber como `token`.
> Cada servicio valida la key por su cuenta contra jota-db. Ver PR #17.

> **Nota extra — PR #6**: Se mergeó también el manejo de fallos del transcriber (watchdog de silencio,
> detección de caída inesperada, notificación al cliente con `service_status`). No tenía issue asociada,
> implementa el spec aprobado en `docs/superpowers/specs/2026-03-14-transcriber-failure-handling-design.md`.

---

### jota-gateway — Issues de protocolo del transcriber (nuevas, descubiertas en análisis)

Descubiertas tras auditar el protocolo completo de jota-transcriber contra nuestra implementación:

| Issue | Título | Prioridad |
|---|---|---|
| #9 | Eliminar `publish_mqtt` del handshake (campo muerto) | P3 |
| #10 | `TranscriberMessage` sin campo `code` — se pierden códigos de error | P2 |
| #11 | Warning `buffer_full` descartado silenciosamente | P2 |
| #12 | `TranscriberConfig` definido en schemas pero no se usa | P3 |
| #13 | `session_id` del mensaje `ready` se pierde | P3 |
| #14 | Idioma hardcodeado a `"es"` — no se lee del handshake del cliente | P2 |
| #15 | `vad_thold` hardcodeado a `0.0` — debería ser configurable | P3 |

---

### jota-transcriber — Issue #27 (race condition, descubierta en análisis)

- [ ] Investigar y corregir race condition entre `flushLoop` y `handleEnd` en `src/server/StreamingSession.h`
  - `handleEnd()` no establece `flush_running_ = false` antes de procesar el final
  - El `flushLoop` puede ejecutar un ciclo más después del `is_final=true` y emitir otro duplicado
  - Fix sugerido: `flush_running_ = false` + `flush_thread_.join()` al inicio de `handleEnd()`
  - Ver análisis detallado en la issue

---

## Fase 1 — Identity + Config: jota-db como servicio de primer nivel

Fusiona el antiguo diseño de "Fase 1 (UUID)" + "Fase 2 (ClientConfig)". Son la misma iniciativa: el gateway resuelve `client_key → Client + ClientConfig` en el handshake con una sola llamada.

> Los servicios internos (orchestrator, transcriber, speaker) siguen validando por su cuenta (zero-trust).
> El gateway **no los verifica** — solo verifica al cliente externo.

**Avance realizado (PR #17):**
- ✅ `client_key` añadido al schema `Handshake`
- ✅ `{client_id}` eliminado del path → `/ws/stream`
- ✅ `client_key` propagado como `x-client-key` al orchestrator y como `token` al transcriber
- ✅ `ORCHESTRATOR_API_KEY` eliminado de config

### Paso 1 — Extender jota-db

- [ ] **jota-db** — Nuevo modelo `ClientConfig` en `src/core/models.py`:
  - Campos: `stt_language`, `stt_model`, `stt_vad_thold`, `tts_voice`, `tts_speed`, `preferred_model_id`, `system_prompt_extra`, `barge_in_enabled`, `barge_in_min_chars`, `conversation_memory_limit`
- [ ] **jota-db** — Migración de base de datos para `ClientConfig`
- [ ] **jota-db** — Auto-crear `ClientConfig` con defaults al crear un `Client`
- [ ] **jota-db** — Nuevo router `src/api/routers/config.py`:
  - `GET  /config/me` → ClientConfig del cliente autenticado
  - `PUT  /config/me` → actualización parcial
  - `POST /config/me/reset` → restaura defaults
- [ ] **jota-db** — Nuevo endpoint `GET /auth/session` en `src/api/routers/auth.py`:
  - Devuelve `{ client: Client, config: ClientConfig }` en una sola llamada
  - Auto-crea `ClientConfig` si el cliente no tiene uno aún

### Paso 2 — DbClient en el gateway

- [ ] **jota-gateway** — Añadir `JOTA_DB_BASE_URL` y `JOTA_DB_API_KEY` a `config.py` y `.env.sample`
- [ ] **jota-gateway** — Crear `src/services/db_client.py` con `DbClient` (singleton compartido):
  - `get_session(client_key)` → `(Client, ClientConfig)` — usado en el handshake WS
  - `get_config(client_id)` → `ClientConfig`
  - `update_config(client_id, patch)` → `ClientConfig`
  - `reset_config(client_id)` → `ClientConfig`
  - `get_conversations(client_id)` → lista
  - `get_messages(conversation_id)` → lista
- [ ] **jota-gateway** — En `routes.py`, tras handshake, llamar `DbClient.get_session(client_key)`:
  - Si 401/403 → cerrar WS con código `1008` (key inválida)
  - Si error de red → cerrar WS con código `1011` (servicio no disponible)
- [ ] **jota-gateway** — `JotaBridge` recibe `client: Client` + `config: ClientConfig` completos
- [ ] **jota-gateway** — Orchestrator recibe `user_id = client.id` (UUID) → FK válida en jota-db (resuelve jota-orchestrator #9)
- [ ] **jota-gateway** — Transcriber recibe `language = config.stt_language` (resuelve issue #14)
- [ ] **jota-gateway** — Transcriber recibe `vad_thold = config.stt_vad_thold` (resuelve issue #15)

---

## Fase 2 — Gateway REST API pública (tras Fase 1)

Con `DbClient` disponible, el gateway expone sus propios endpoints REST al cliente externo.
Auth: `X-API-Key: {client_key}` → dependency `get_verified_client()`.

- [ ] **jota-gateway** — Crear dependency `get_verified_client(x_api_key)` → llama `DbClient.get_session()`
- [ ] **jota-gateway** — Crear `src/api/http_routes.py` con:
  - `GET    /api/config` — lee ClientConfig (via DbClient)
  - `PUT    /api/config` — actualiza campos parcialmente (via DbClient)
  - `POST   /api/config/reset` — restaura defaults (via DbClient)
  - `GET    /api/conversations` — lista conversaciones (via DbClient)
  - `GET    /api/conversations/{id}/messages` — mensajes (via DbClient)
  - `DELETE /api/conversations/{id}` — archiva (via DbClient)
  - `GET    /api/models` — modelos disponibles (proxy a orchestrator)
  - `GET    /api/health` — estado de todos los servicios internos
- [ ] **jota-gateway** — Montar el router en `main.py`

---

## Fase 3 — Propagación de ClientConfig a los servicios (tras Fase 1)

El gateway ya tiene `Client` + `ClientConfig` en `JotaBridge` desde Fase 1.
Inyectar esos valores al conectar con cada servicio interno.

- [ ] **jota-gateway** — TTS handshake usa `config.tts_voice` y `config.tts_speed`
- [ ] **jota-gateway** — Orchestrator payload incluye `model_id = config.preferred_model_id`
- [ ] **jota-gateway** — Orchestrator payload incluye `system_extra = config.system_prompt_extra`
- [ ] **jota-gateway** — `BARGE_IN_MIN_CHARS` global sustituido por `config.barge_in_min_chars` por cliente
