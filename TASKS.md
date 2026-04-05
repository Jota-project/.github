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
