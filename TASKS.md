# Jota — Issues pendientes

> Actualizado: 2026-07-01. Ver arquitectura completa en [`ARCHITECTURE.md`](ARCHITECTURE.md).

---

## Estado del sistema

El sistema está alineado con la nueva arquitectura modular:

- **jota-gateway** — BFF con SQLite local para identidad/clientes/config. Cuatro superficies: WS, Admin REST, OpenAI-compat, Health.
- **jota-transcriber** — Whisper streaming C++17. Auth estática por defecto.
- **jota-speaker** — Kokoro TTS + Wyoming protocol para Home Assistant.
- **OpenClaw** — Orquestador LLM externo (camino Maintained).
- **jota-orchestrator + jota-inference** — Camino Alternative 100% local; menos desarrollo futuro.
- **jota-db** — Deprecated como fuente de identidad; mantenido como auth externa opcional.

---

## Issues activas

### jota-gateway

#### #52 — `/v1/*` auth model — security/compatibility conflict

Endpoints OpenAI-compat expuestos externamente sin auth, requerido por Home Assistant. Decidir entre LAN-only (nginx) o API key obligatoria. Documentar la decisión final en `ARCHITECTURE.md`.

#### #50 — E2E test suite with Docker Compose

Suite E2E contra microservicios reales en Docker. Útil para detectar regresiones de contratos entre gateway ↔ transcriber/speaker.

#### #49 — `DELETE /api/conversations/{id}` — archive endpoint

Pendiente de implementar.

#### #48 — Cachear `get_verified_client()` para evitar round-trip a SQLite por request

Pendiente. TTL propuesto: 60 s.

#### (nueva) — Deprecar jota-db como auth backend en transcriber/speaker

Migrar transcriber y speaker a `AUTH_TOKEN` estático per-service por defecto. `AUTH_API_URL` queda como opción para setups que aún quieran centralizar auth.

Issue abierta: https://github.com/Jota-project/jota-gateway/issues/78

### jota-transcriber

#### #27 — Race condition `flushLoop`/`handleEnd` → duplicados `is_final`

`handleEnd()` no establece `flush_running_ = false` antes de procesar el final. Fix sugerido: `flush_running_ = false` + `flush_thread_.join()` al inicio de `handleEnd()`. Ver análisis en `src/server/StreamingSession.h:380-416, 471-556`.

---

## Changelog reciente

- **#67 (jota-gateway)** — Reemplazar jota-db con SQLite local.
- **#66 (jota-gateway)** — API redesign: typed WS protocol, admin routes, health endpoints.
- **#64 (jota-gateway)** — OpenClaw multiplexed concurrent sessions + agent-initiated push delivery.
- **#51 (jota-gateway)** — Eliminar `GET /models` redundante.
- **#18 (jota-orchestrator)** — `QuickRequest` acepta `system_prompt_extra`.
