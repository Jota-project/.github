# Jota — Issues pendientes

> Actualizado: 2026-04-05. Ver arquitectura completa en `ARCHITECTURE.md`.

---

## Estado del sistema

El sistema está funcionalmente completo en su arquitectura BFF:

- **jota-gateway** — BFF con WebSocket + REST API. ClientConfig propagada a todos los servicios.
- **jota-db** — Fuente de verdad. ClientConfig implementada y autoasignada por cliente.
- **jota-orchestrator** — `system_prompt_extra`, UUID real como `client_id`, memoria funcional.
- **jota-transcriber** — Auth por API, parciales en tiempo real, manejo de fallos.
- **jota-speaker** — Auth contra jota-db, TTS streaming.

---

## Feature en desarrollo: ServiceConfig + InferenceProvider

> Spec completa: `docs/superpowers/specs/2026-04-08-serviceconfig-providers-design.md`

Añade una capa de configuración centralizada en jota-db y soporte para múltiples backends de IA.

### jota-db — Prioridad P1

**Tarea 1: Modelo `ServiceConfig`**
- Nueva tabla `service_config (service, key, value JSON, description)`
- Lógica de seeding al arrancar desde variables de entorno
- Variables de seeding: `SEED_TRANSCRIBER_MODEL`, `SEED_SPEAKER_MODEL`, `SEED_ORCHESTRATOR_DEFAULT_PROVIDER`, etc.

**Tarea 2: Endpoints `/admin/config/*`**
- `GET /admin/config` — listar toda la config
- `GET /admin/config/{service}` — config de un servicio
- `PUT /admin/config/{service}/{key}` — crear/actualizar valor
- `DELETE /admin/config/{service}/{key}` — eliminar valor
- `POST /admin/config/{service}/reset` — restaurar defaults desde `.env`
- Auth: mismo mecanismo Bearer existente

**Tarea 3: Modelo `InferenceProvider`**
- Nueva tabla `inference_provider` con campos: `id, name, type, base_url, api_key, default_model_id, is_active, extra_config`
- Types: `local | openai | anthropic | custom`
- Seeding del provider local desde `.env` al arrancar

**Tarea 4: Endpoints `/admin/providers/*`**
- `GET /admin/providers` — listar providers
- `POST /admin/providers` — crear provider
- `GET /admin/providers/{id}` — detalle
- `PUT /admin/providers/{id}` — actualizar
- `DELETE /admin/providers/{id}` — desactivar (soft delete)

### jota-gateway — Prioridad P1

**Tarea 5: Proxy `/api/admin/*`**
- Añadir cliente HTTP para los nuevos endpoints admin de jota-db
- Exponer `GET/PUT/DELETE /api/admin/config/{service}/{key}`
- Exponer `GET/POST/PUT/DELETE /api/admin/providers/{id?}`
- Auth: `X-API-Key` requerido (igual que resto de API)

### jota-inference — Prioridad P1 — issue #23

**Tarea 6: API HTTP compatible OpenAI**
- Servidor HTTP en puerto configurable (`INFERENCE_HTTP_PORT`, default `8003`)
- `POST /v1/chat/completions` con streaming SSE
- `GET /v1/models`
- Auth Bearer (mismo token que WebSocket actual)
- Prefix caching interno para reutilizar KV-cache (fase 2)
- El WebSocket actual puede coexistir durante la transición

### jota-orchestrator — Prioridad P1 (después de jota-db + jota-inference) — issue #21

**Tarea 7: Capa `LLMProvider`**
- Interfaz abstracta `LLMProvider` con `generate(messages, ...) → AsyncIterator` y `list_models()`
- API **stateless**: el caller pasa el historial completo, sin gestión de sesiones
- Implementaciones: `OpenAICompatibleProvider` (jota-inference HTTP, VPS, Ollama), `OpenAIProvider`, `AnthropicProvider`
- Al arrancar: leer `orchestrator.default_provider_id` desde `GET /admin/config/orchestrator`
- Migrar `quick.py`, `controller/input.py`, `controller/models.py`, `chat/websocket.py`
- Eliminar session management (`create/set_context/close_session`) del orquestador
- Selección: `ClientConfig.preferred_model_id` > `default_provider_id`
- Cachear config de provider con TTL 5 min

---

## Issues abiertas

### jota-gateway — #8 — Crear suite de tests y CI

La única issue de trabajo real. Abarca:

- Tests unitarios para `DbClient`, `OrchestratorClient`, `TranscriberClient`, `TTSClient`
- Tests de integración para los endpoints REST (`/api/config`, `/api/conversations`, `/api/models`, `/api/health`)
- Tests del WebSocket BFF (`/ws/stream`) con mocks de los servicios internos
- Pipeline CI en GitHub Actions (lint + test en cada PR)

### jota-orchestrator — #15 — Eliminar `GET /models`

`GET /models` en el orchestrator es un proxy redundante a jota-db. Los clientes ahora usan `GET /api/models` del gateway. Limpieza técnica, no bloqueante.

Verificar que ningún cliente externo llame al orchestrator directamente antes de eliminar.

### jota-db — #12 — Revisar y cerrar

`ClientConfig` ya está implementada (#11). Revisar si queda algo pendiente en esta issue o cerrarla.

### jota-transcriber — #27 — Race condition `flushLoop`/`handleEnd`

`handleEnd()` no establece `flush_running_ = false` antes de procesar el final. El `flushLoop` puede ejecutar un ciclo más y emitir un duplicado `is_final=true`.

Fix sugerido: `flush_running_ = false` + `flush_thread_.join()` al inicio de `handleEnd()`.

Ver análisis en `src/server/StreamingSession.h:380-416, 471-556`.
