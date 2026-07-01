# Jota Docs Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Alinear la documentación raíz de la organización `Jota-project` y los READMEs de los 6 repos de servicio con la arquitectura vigente (gateway con SQLite local + OpenClaw, clientes `jota-voice`/`jota-display` transferidos desde la cuenta personal, repos legacy marcados como Alternative/Deprecated).

**Architecture:** Cambios puramente documentales (markdown), ejecutados como 8 PRs independientes — uno por repo — más un paso de verificación final de los transfers del usuario. Cada PR modifica como mucho un README + opcionalmente un archivo nuevo de soporte (`docs/legacy-*.md`). Sin cambios funcionales de código.

**Tech Stack:** Markdown, Git, GitHub CLI (`gh`) para crear PRs e issues.

---

## Global Constraints

- **Solo docs.** Ningún task modifica código de producto, dependencias, Docker, ni configuración runtime.
- **Badge de estado coherente.** Todo repo de la org lleva uno de: `Maintained`, `Alternative`, `Deprecated` — definido en el spec §3.1.
- **Naming del banner legacy.** Patrón literal: *"Este repo está en modo Alternative/Deprecated. El camino recomendado es OpenClaw (ver [Jota-project/.github](https://github.com/Jota-project/.github/blob/main/README.md)). Mantenemos el código por compatibilidad."*
- **Tono y voz.** Coherente con el estilo actual del gateway v1.9.0: directo, segunda persona ocasional, sin marketing. Sin emojis decorativos en banners; permitidos en tablas y listas donde ya existan.
- **Commits pequeños y frecuentes.** Un commit por cambio lógico. Mensajes en español o inglés indistintamente; seguir convención del repo destino si la tiene (ver §5 del spec).
- **PRs por repo, no por archivo.** Si un repo necesita cambios en dos archivos (README + `docs/legacy-*.md`), van en el mismo PR.
- **No se hace push sin OK explícito del usuario.** Cada PR se crea pero no se mergea hasta revisión.
- **Branches `feat/`** siguiendo `CONTRIBUTING.md` actual (punto "Pull Request Process").
- **Rama base de cada repo:**
  - `jota-gateway` → `main` (asumido; verificar antes del PR)
  - `jota-orchestrator` → `main`
  - `jota-inference` → `master` ⚠️ (no `main`)
  - `jota-db` → `main` (clon local apunta a `SitoSt/JotaDB`; el repo de la org `Jota-project/jota-db` debe existir — verificar antes de clonar)
  - `jota-transcriber` → `main`
  - `jota-speaker` → `main`

---

## Task Map (resumen)

| # | Repo | Acción | PR |
|---|---|---|---|
| 1 | `Jota-project/.github` | Mover `profile/README.md` → `README.md`, reescribir `ARCHITECTURE.md`, `TASKS.md`, `CONTRIBUTING.md` | PR-1 |
| 2 | `Jota-project/jota-speaker` | Reencuadrar README con contexto de org | PR-2 |
| 3 | `Jota-project/jota-orchestrator` | Marcar como Alternative + reescritura media | PR-3 |
| 4 | `Jota-project/jota-inference` | Marcar como Alternative + reescritura fuerte | PR-4 |
| 5 | `Jota-project/jota-db` | Marcar como Deprecated + mover Tasks/Events a legacy doc | PR-5 |
| 6 | `Jota-project/jota-transcriber` | Edición quirúrgica del README | PR-6 |
| 7 | `Jota-project/jota-gateway` | Issue nueva (deprecar jota-db como auth backend) | Issue-1 |
| 8 | `Jota-project/jota-voice` + `Jota-project/jota-display` | Post-transfer: verificar URLs y referenciar desde `.github` | PR-7, PR-8 |
| 9 | Verificación final | Cruzar links, comprobar badges, cerrar criterios de done | — |

---

### Task 1: Reescribir repo raíz `Jota-project/.github`

**Files:**
- Modify: `/Users/alfonsogarre/Workspace/jota_.github/README.md` (mover contenido de `profile/README.md` aquí, ampliar)
- Delete: `/Users/alfonsogarre/Workspace/jota_.github/profile/` (carpeta completa)
- Modify: `/Users/alfonsogarre/Workspace/jota_.github/ARCHITECTURE.md`
- Modify: `/Users/alfonsogarre/Workspace/jota_.github/TASKS.md`
- Modify: `/Users/alfonsogarre/Workspace/jota_.github/CONTRIBUTING.md`

**Working dir:** `/Users/alfonsogarre/Workspace/jota_.github`

**Branching:**
- [ ] **Step 1.1: Crear branch**

```bash
cd /Users/alfonsogarre/Workspace/jota_.github
git checkout -b feat/docs-alignment-root
```

**README.md (raíz):**
- [ ] **Step 1.2: Mover `profile/README.md` a `README.md` raíz y ampliar**

```bash
mv profile/README.md README.md
rm -rf profile/
```

Luego editar `README.md` (en la raíz) para añadir/ajustar:

```markdown
# Jota — Distributed Voice AI Assistant

> A modular, privacy-first voice assistant built for local execution. One gateway, specialized microservices, OpenAI-compatible.

Jota is a full-stack AI assistant ecosystem. A physical client (ESP32, web, app, Termux, Home Assistant) connects to a single entry point — the **gateway** — which resolves identity, loads per-client configuration, and coordinates real-time speech, LLM, and synthesis across specialized services.

---

## Services

| Repo | Language | Role | Status |
| :--- | :--- | :--- | :--- |
| **[jota-gateway](https://github.com/Jota-project/jota-gateway)** | Python | BFF — single entry point, auth, REST API, voice session orchestration | `Maintained` |
| **[jota-transcriber](https://github.com/Jota-project/jota-transcriber)** | C++17 | Real-time Whisper STT over WebSocket | `Maintained` |
| **[jota-speaker](https://github.com/Jota-project/jota-speaker)** | Python | Streaming TTS (Kokoro) + Wyoming protocol for Home Assistant | `Maintained` |
| **[jota-orchestrator](https://github.com/Jota-project/jota-orchestrator)** | Python | LLM routing, tool calling, conversation memory | `Alternative` |
| **[jota-inference](https://github.com/Jota-project/jota-inference)** | C++ | Local LLM execution via llama.cpp | `Alternative` |
| **[jota-db](https://github.com/Jota-project/jota-db)** | Python | Centralized auth for legacy service setups | `Deprecated` |

### Alternative vs Maintained

The **Maintained** path is the recommended one — the gateway uses local SQLite for identity and configuration, and **OpenClaw** as the LLM orchestrator (external open-source project).

The **Alternative** path (`jota-orchestrator` + `jota-inference`) keeps Jota 100% self-hosted with first-party Python/C++ services. It receives less active development; new deployments should prefer the Maintained path unless full local control of the inference layer is a hard requirement.

The **Deprecated** path (`jota-db`) is kept for compatibility with setups that still use it as a centralized auth backend. New code should use per-service `AUTH_TOKEN` or each service's own storage.

See [`ARCHITECTURE.md`](ARCHITECTURE.md) for the full architecture map and [`TASKS.md`](TASKS.md) for active work.

---

## Clients

| Repo | Description |
| :--- | :--- |
| **[jota-voice](https://github.com/Jota-project/jota-voice)** | Termux/Android client. Replaces wyoming-satellite with direct WS streaming to the gateway. |
| **[jota-display](https://github.com/Jota-project/jota-display)** | Always-on kiosk UI (Vue 3 + SSE) showing voice session state and home controls. |

Both clients run on the same Android device and connect to the gateway over LAN.

### Personal experiments

The author also maintains auxiliary tooling outside this organization. These are not part of the Jota stack but may be useful:

- [SitoSt/jota-wake-trainer](https://github.com/SitoSt/jota-wake-trainer) — CLI to train custom wake words with openWakeWord.
- [SitoSt/jota-dashboard](https://github.com/SitoSt/jota-dashboard) — Web dashboard.

---

## Quick start

Clone the maintained repositories:

```bash
git clone https://github.com/Jota-project/jota-gateway.git
git clone https://github.com/Jota-project/jota-transcriber.git
git clone https://github.com/Jota-project/jota-speaker.git
# Plus OpenClaw running separately (see OpenClaw docs)
# Plus a client: jota-voice or jota-display
```

Then follow each repo's README for environment variables and Docker compose instructions. The gateway README is the entry point.

---

*Built for local deployment. Designed to grow.*
```

- [ ] **Step 1.3: Reescribir `ARCHITECTURE.md`**

Reemplazar todo el contenido de `/Users/alfonsogarre/Workspace/jota_.github/ARCHITECTURE.md` con:

```markdown
# Jota — System Architecture

> Última actualización: 2026-07-01

---

## 1. Visión General

Jota es un asistente de voz distribuido. Un cliente físico (ESP32, app, web, Home Assistant, Termux) habla con un único punto de entrada (gateway), que orquesta varios microservicios especializados.

### Mapa de servicios

\`\`\`
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
\`\`\`

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

\`\`\`
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
\`\`\`

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
\`\`\`json
{ "type": "config", "language": "es", "token": "<client_key>", "vad_thold": 0.0 }
\`\`\`

**Mensajes del servidor:** `ready`, `transcription` (con `is_final`), `warning` (code `buffer_full`), `error`. Fin de sesión: `{"type":"end"}`.

**Auth del transcriber:**
- Default: `AUTH_TOKEN=<token>` estático.
- Opcional: `AUTH_API_URL=<external URL>` + `AUTH_API_SECRET` para validar contra un backend externo. **Nota:** la opción recomendada hoy es `AUTH_TOKEN` estático; ver [issue abierta](https://github.com/Jota-project/jota-gateway/issues) sobre deprecar el uso de `jota-db` como auth backend.

### 5.2 jota-speaker

**WebSocket** (`/ws`):
\`\`\`json
{ "type": "auth", "token": "<TTS_TOKEN>", "voice": "ef_dora", "speed": 1.0 }
\`\`\`

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
```

- [ ] **Step 1.4: Reescribir `TASKS.md`**

Reemplazar todo el contenido de `/Users/alfonsogarre/Workspace/jota_.github/TASKS.md` con:

```markdown
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
```

- [ ] **Step 1.5: Actualizar `CONTRIBUTING.md`**

Reemplazar el contenido de `/Users/alfonsogarre/Workspace/jota_.github/CONTRIBUTING.md` con:

```markdown
# Contributing to Jota 🚀

First off, thank you for considering contributing to Jota!

### Estado de los repos

Antes de abrir un PR, mira el [`README.md`](README.md) raíz y [`ARCHITECTURE.md`](ARCHITECTURE.md) para entender qué servicios están `Maintained`, `Alternative` o `Deprecated`. Los PRs van primero contra los repos `Maintained`.

### 🧪 Development Workflow

1. **Docker is King:** Every service has a `Dockerfile`. Ensure your changes don't break the build:
   `docker build -t jota-service-check .`
2. **Linting:**
   * Python: use `ruff` or `black`.
   * C++: follow the existing Clang-Format style.
3. **Tests:** Run tests before submitting a PR.

### 📬 Pull Request Process

* Create a feature branch (`feat/your-feature`).
* Include tests for new logic.
* Update the relevant `README.md` if parameters change.
* For doc-only changes, one PR per repo is the standard.

### 📚 Documentación

Las decisiones arquitectónicas grandes van en `ARCHITECTURE.md`. Cambios pequeños van en el README del repo específico. Specs de diseño van en `docs/superpowers/specs/`.
```

- [ ] **Step 1.6: Verificar que no quedan referencias a `profile/`**

```bash
cd /Users/alfonsogarre/Workspace/jota_.github
grep -r "profile/" --include="*.md" --include="*.yml" --include="*.yaml" . 2>/dev/null
```

Expected: cero resultados.

- [ ] **Step 1.7: Commit**

```bash
cd /Users/alfonsogarre/Workspace/jota_.github
git add README.md ARCHITECTURE.md TASKS.md CONTRIBUTING.md
git status
git commit -m "docs: align org-root docs with current architecture (jul 2026)

- Move profile/README.md → README.md (org landing page convention)
- Rewrite ARCHITECTURE.md with Maintained/Alternative/Deprecated split
- Refresh TASKS.md with current open issues + jota-db deprecation note
- Update CONTRIBUTING.md with repo status guidance

Refs: docs/superpowers/specs/2026-07-01-jota-docs-alignment-design.md

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 1.8: Push y abrir PR**

```bash
cd /Users/alfonsogarre/Workspace/jota_.github
git push -u origin feat/docs-alignment-root
gh pr create --base main --head feat/docs-alignment-root \
  --title "docs: align org-root docs with current architecture" \
  --body "Reescribe los docs raíz (.github) para reflejar la arquitectura vigente: gateway con SQLite local + OpenClaw como camino Maintained, jota-orchestrator/inference como Alternative, jota-db como Deprecated. Mueve \`profile/README.md\` a \`README.md\` raíz (convención actual de GitHub).

Spec: docs/superpowers/specs/2026-07-01-jota-docs-alignment-design.md"
```

Expected: PR abierto. NO mergear — esperar revisión del usuario.

---

### Task 2: Reencuadrar README de `jota-speaker`

**Files:**
- Modify: `/Users/alfonsogarre/Workspace/jota-speaker/README.md`

**Working dir:** `/Users/alfonsogarre/Workspace/jota-speaker`

- [ ] **Step 2.1: Crear branch**

```bash
cd /Users/alfonsogarre/Workspace/jota-speaker
git checkout -b feat/docs-org-context
```

- [ ] **Step 2.2: Insertar bloque al inicio del README, antes del H1 actual**

Abrir `README.md` y como **primeras líneas del archivo** (antes del `# jota-speaker`), añadir:

```markdown
![Status: Maintained](https://img.shields.io/badge/status-Maintained-2ea44f)

> **Role in Jota ecosystem:** TTS streaming microservice. WebSocket receives LLM tokens and emits PCM16 audio; Wyoming TCP (port 20424) lets Home Assistant use it as native TTS. The gateway `jota-gateway` connects to this service per voice session.
>
> Part of [Jota-project](https://github.com/Jota-project). See [`ARCHITECTURE.md`](https://github.com/Jota-project/.github/blob/main/ARCHITECTURE.md) for the full system map.

---
```

(El bloque va seguido del contenido actual del README tal cual. No se borra nada de lo existente.)

- [ ] **Step 2.3: Reordenar sección Wyoming — subirla como §6 destacada**

Buscar la sección existente "Wyoming protocol (Home Assistant)" (actualmente al final del README). Cortar y pegar como nueva **§6**, justo después de "Configuration". Renumerar las secciones siguientes si las hay.

En la nueva §6, anteponer este párrafo introductorio:

```markdown
> **Why this matters:** if you're integrating Jota with Home Assistant, this is the path you'll use. Wyoming is the protocol HA's voice pipeline speaks natively. jota-speaker implements both the WebSocket surface (for the gateway) and the Wyoming TCP surface (for HA) from a single process — no extra service needed.
```

- [ ] **Step 2.4: Añadir sección "Status & roadmap" al final, antes del cierre del README**

```markdown
## Status & roadmap

- **Status:** Maintained. Most recent work added Wyoming protocol support (PR landed 2026-05-21).
- **Default voice:** `ef_dora` (Spanish, female). Configurable via `JOTA_KOKORO_VOICE`.
- **Active directions:**
  - Auth migration: planning to move from `jota-db` external auth to per-service `TTS_TOKEN` (tracked in [`Jota-project/jota-gateway` issue tracker](https://github.com/Jota-project/jota-gateway/issues)).
  - Wyoming: protocol coverage for HA discoverability is complete; expect incremental fixes as HA updates.
```

- [ ] **Step 2.5: Commit**

```bash
cd /Users/alfonsogarre/Workspace/jota-speaker
git add README.md
git commit -m "docs: add org context, lead with Wyoming for HA users

Refs: docs/superpowers/specs/2026-07-01-jota-docs-alignment-design.md

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 2.6: Push y abrir PR**

```bash
cd /Users/alfonsogarre/Workspace/jota-speaker
git push -u origin feat/docs-org-context
gh pr create --base main --head feat/docs-org-context \
  --title "docs(speaker): add org context and lead with Wyoming section" \
  --body "Añade badge Maintained, contexto de org, reordena la sección Wyoming para que destaque al inicio, y añade sección Status & roadmap. Sin cambios funcionales."
```

---

### Task 3: Reencuadrar README de `jota-orchestrator` (Alternative)

**Files:**
- Modify: `/Users/alfonsogarre/Workspace/jota-orchestrator/README.md`

**Working dir:** `/Users/alfonsogarre/Workspace/jota-orchestrator`

- [ ] **Step 3.1: Crear branch**

```bash
cd /Users/alfonsogarre/Workspace/jota-orchestrator
git checkout -b feat/docs-mark-alternative
```

- [ ] **Step 3.2: Insertar banner al inicio del README, antes del primer H1 existente**

Abrir `README.md`. Como **primeras líneas** (antes del H1), añadir:

```markdown
![Status: Alternative](https://img.shields.io/badge/status-Alternative-yellow)

> ⚠️ **Este repo está en modo Alternative.** El camino recomendado es [OpenClaw](https://github.com/Jota-project/.github/blob/main/ARCHITECTURE.md) (orquestador externo open source). Mantenemos este código por compatibilidad con setups 100% locales que usan `jota-orchestrator` + `jota-inference` como camino LLM propio. Recibe parches solo para issues críticas; los PRs nuevos se priorizan contra los repos `Maintained` (ver [org README](https://github.com/Jota-project/.github)).
>
> Si estás empezando un deploy nuevo, usa OpenClaw + `llama.cpp` standalone.

---
```

(El resto del README existente se conserva tal cual — variables de entorno, endpoints, testing siguen siendo válidos para quien mantenga este camino.)

- [ ] **Step 3.3: Reescribir la sección de variables de entorno obsoleta**

Buscar la sección `### Variables de Entorno` (o equivalente). Eliminar la línea:

```
JOTA_DB_URL="http://localhost:8080"
JOTA_DB_SK="tu_server_key"
```

Reemplazar por una nota:

```markdown
### Variables de Entorno

> **Nota:** `JOTA_DB_URL` y `JOTA_DB_SK` ya no se usan. El gateway ahora resuelve identidad vía SQLite local y propaga `x-client-key` + `x-client-id` por header al orquestador. Si mantienes un setup legacy, consulta [`Jota-project/jota-db`](https://github.com/Jota-project/jota-db).
```

- [ ] **Step 3.4: Añadir sección "Status & roadmap" al final**

```markdown
## Status & roadmap

- **Status:** Alternative. Frozen on `1.1.0` (2026-04-05).
- **No new feature work planned.** The recommended path is OpenClaw. Maintenance is limited to critical fixes.
- **For new deployments:** use OpenClaw + llama.cpp. See [org ARCHITECTURE.md](https://github.com/Jota-project/.github/blob/main/ARCHITECTURE.md).
```

- [ ] **Step 3.5: Commit**

```bash
cd /Users/alfonsogarre/Workspace/jota-orchestrator
git add README.md
git commit -m "docs: mark as Alternative and remove obsolete JOTA_DB_URL guidance

Refs: docs/superpowers/specs/2026-07-01-jota-docs-alignment-design.md

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 3.6: Push y abrir PR**

```bash
cd /Users/alfonsogarre/Workspace/jota-orchestrator
git push -u origin feat/docs-mark-alternative
gh pr create --base main --head feat/docs-mark-alternative \
  --title "docs(orchestrator): mark as Alternative path, remove obsolete JOTA_DB env vars" \
  --body "Marca el repo como Alternative y elimina la guía obsoleta de JOTA_DB_URL. El camino recomendado es OpenClaw."
```

---

### Task 4: Reencuadrar README de `jota-inference` (Alternative, rama `master`)

**Files:**
- Modify: `/Users/alfonsogarre/Workspace/jota-inference/README.md`

**Working dir:** `/Users/alfonsogarre/Workspace/jota-inference` ⚠️ **rama `master`, no `main`**

- [ ] **Step 4.1: Crear branch desde master**

```bash
cd /Users/alfonsogarre/Workspace/jota-inference
git branch --show-current  # verificar que estamos en master
git checkout -b feat/docs-mark-alternative
```

- [ ] **Step 4.2: Insertar banner al inicio del README**

Abrir `README.md`. Como **primeras líneas** (antes del H1 existente), añadir:

```markdown
![Status: Alternative](https://img.shields.io/badge/status-Alternative-yellow)

> ⚠️ **Este repo está en modo Alternative.** Sin commits desde marzo 2026. Para nuevos deployments, recomendamos usar [`llama.cpp`](https://github.com/ggerganov/llama.cpp) standalone combinado con [OpenClaw](https://github.com/Jota-project/.github/blob/main/ARCHITECTURE.md) en lugar de `jota-orchestrator`. Este binario se mantiene por compatibilidad con setups existentes que usan `jota-orchestrator` + `jota-inference`.
>
> Si llegas aquí desde el [Jota-project](https://github.com/Jota-project), consulta el [org README](https://github.com/Jota-project/.github) para entender el rol de cada repo en el ecosistema.

---
```

(El contenido técnico existente — Quick Start, WebSocket API, ejemplos de cliente — se conserva tal cual. Sigue siendo válido para quien use este binario.)

- [ ] **Step 4.3: Editar la sección de auth para dejar JotaDB como opción**

Buscar la sección `### 2. Configure Authentication`. Cambiar el segundo párrafo para que quede:

```markdown
### 2. Configure Authentication

The server supports two authentication modes:

**Static token (recommended for simple deployments):**
\`\`\`bash
export AUTH_TOKEN="your_secure_token_here"
\`\`\`

**External auth via JotaDB (legacy setups):**
The server can validate tokens against a central `jota-db` instance:
\`\`\`bash
export JOTA_DB_URL="http://localhost:8000/api/db"
\`\`\`
This mode is kept for compatibility with setups that centralize auth across multiple Jota services. For new deployments, prefer the static `AUTH_TOKEN` mode.
```

- [ ] **Step 4.4: Añadir sección "Status & roadmap" al final**

```markdown
## Status & roadmap

- **Status:** Alternative. Frozen on `1.1.0` (2026-03-16).
- **No new feature work planned.** For new local LLM deployments, use `llama.cpp` standalone with OpenClaw.
- **For new deployments:** see [org ARCHITECTURE.md](https://github.com/Jota-project/.github/blob/main/ARCHITECTURE.md) for the recommended stack.
```

- [ ] **Step 4.5: Commit**

```bash
cd /Users/alfonsogarre/Workspace/jota-inference
git add README.md
git commit -m "docs: mark as Alternative and clarify llama.cpp + OpenClaw as recommended

Refs: docs/superpowers/specs/2026-07-01-jota-docs-alignment-design.md

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 4.6: Push y abrir PR contra `master`**

```bash
cd /Users/alfonsogarre/Workspace/jota-inference
git push -u origin feat/docs-mark-alternative
gh pr create --base master --head feat/docs-mark-alternative \
  --title "docs(inference): mark as Alternative, recommend llama.cpp + OpenClaw" \
  --body "Marca el repo como Alternative y aclara que la opción recomendada para nuevos deployments es llama.cpp standalone + OpenClaw. La sección de auth ahora distingue entre AUTH_TOKEN estático (recomendado) y JotaDB (legacy)."
```

---

### Task 5: Marcar `jota-db` como Deprecated + mover Tasks/Events a legacy doc

**Files:**
- Modify: `/Users/alfonsogarre/Workspace/JotaDB/README.md` (rename a `jota-db` si transfieres primero — ver §8)
- Create: `/Users/alfonsogarre/Workspace/JotaDB/docs/legacy-tasks-api.md`

**Working dir:** `/Users/alfonsogarre/Workspace/JotaDB` ⚠️ **el remote apunta a `SitoSt/JotaDB`, no a `Jota-project/jota-db`. Verificar antes de pushear.**

- [ ] **Step 5.1: Verificar el remote y rama del clon local**

```bash
cd /Users/alfonsogarre/Workspace/JotaDB
git remote -v
git branch --show-current
```

Si el remote es `SitoSt/JotaDB`, NO pushear a origin. En su lugar:
- Cambiar el remote: `git remote set-url origin https://github.com/Jota-project/jota-db.git`
- `git fetch origin`
- `git checkout -b feat/docs-deprecate origin/main`

Si el repo `Jota-project/jota-db` no existe en GitHub todavía, parar este task y avisar al usuario.

- [ ] **Step 5.2: Crear `docs/legacy-tasks-api.md` con el contenido extraído**

Crear `/Users/alfonsogarre/Workspace/JotaDB/docs/legacy-tasks-api.md` con todo el contenido de la sección "Tasks" y "Events" del README actual (las dos secciones con ejemplos de `curl` que aparecen al principio). Encabezado:

```markdown
# Legacy Tasks & Events API

> ⚠️ **DEPRECATED.** This API surface was part of an earlier iteration of `jota-db`. It is no longer part of the active system and is kept here only for historical reference and to support external integrations that may still call it.

The endpoints documented below (`/tasks`, `/events`) are present in some older deployments but are not part of the current Jota architecture. The current role of `jota-db` is centralized authentication for legacy service setups — see the [main README](../README.md).

---

## Tasks (legacy)
<pegar aquí toda la sección "## 📝 Ejemplos de Uso de la API" → "#### Tareas" del README actual, desde el inicio hasta justo antes de "### Events">

## Events (legacy)
<pegar aquí toda la sección "### Events (Eventos)" del README actual>
```

- [ ] **Step 5.3: Reemplazar `README.md` con versión reenfocada**

Sobreescribir `/Users/alfonsogarre/Workspace/JotaDB/README.md` con:

```markdown
# jota-db

![Status: Deprecated](https://img.shields.io/badge/status-Deprecated-red)

> ⚠️ **Deprecated as identity/config source.** As of `jota-gateway` v1.9.0, the gateway maintains its own SQLite database for client identity and configuration. `jota-db` is kept for compatibility with setups that still use it as a centralized auth backend for `jota-transcriber` and `jota-speaker`.
>
> See [org ARCHITECTURE.md](https://github.com/Jota-project/.github/blob/main/ARCHITECTURE.md) §10 for the full deprecation context.

---

## Current role

`jota-db` provides:

- **Centralized auth API** for legacy service setups (transcriber, speaker, others) via `AUTH_API_URL` + `AUTH_API_SECRET`.
- **Admin REST API** (`/admin/services`, `/admin/clients`, `/admin/providers`, `/admin/config`) for managing `ServiceConfig`, `InferenceProvider`, `AdminUser`, and `Client`/`ClientConfig`.
- **Internal REST API** (`/internal/`) for service-to-service configuration and provider queries.

## Migration direction

The dependency on `jota-db` for client identity is removed in `jota-gateway`. The remaining dependencies (`jota-transcriber` and `jota-speaker` using it as external auth) are tracked in [`Jota-project/jota-gateway` issues](https://github.com/Jota-project/jota-gateway/issues) under the deprecation effort.

If you maintain a fresh deployment:
- **Don't** use `jota-db` as identity source.
- For local auth per service, use `AUTH_TOKEN` static.
- For centralized auth, you can still use `jota-db` until the migration is complete.

## Components

| Component | Path |
| :--- | :--- |
| All data models | `src/core/models.py` |
| Bearer security | `src/api/security.py` |
| Auth endpoints (`/auth/client`, `/auth/validate`) | `src/api/routers/auth.py` |
| Config endpoints (`/config/me`) | `src/api/routers/config.py` |
| Chat (conversations, messages) | `src/api/routers/chat.py` |
| Admin routers (`/admin/services`, `/admin/clients`, `/admin/providers`, `/admin/config`) | `src/api/routers/admin/` |
| Internal routers (`/internal/`) | `src/api/routers/internal/` |

## Historical

The Tasks/Events API documented in early iterations of this repo is no longer part of the active system. See [`docs/legacy-tasks-api.md`](docs/legacy-tasks-api.md) for the historical surface, kept only for reference.
```

- [ ] **Step 5.4: Commit**

```bash
cd /Users/alfonsogarre/Workspace/JotaDB
git add README.md docs/legacy-tasks-api.md
git commit -m "docs: mark as Deprecated, move Tasks/Events to legacy doc

Refs: docs/superpowers/specs/2026-07-01-jota-docs-alignment-design.md

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 5.5: Push y abrir PR**

```bash
cd /Users/alfonsogarre/Workspace/JotaDB
git push -u origin feat/docs-deprecate
gh pr create --base main --head feat/docs-deprecate \
  --title "docs(db): mark as Deprecated, move Tasks/Events to legacy doc" \
  --body "Marca jota-db como Deprecated (gateway ya tiene SQLite local). Mueve la sección Tasks/Events a docs/legacy-tasks-api.md con banner de deprecated. El README principal ahora describe el rol real (auth centralizada + routers admin/internal)."
```

---

### Task 6: Edición quirúrgica del README de `jota-transcriber`

**Files:**
- Modify: `/Users/alfonsogarre/Workspace/jota-transcriber/README.md`

**Working dir:** `/Users/alfonsogarre/Workspace/jota-transcriber`

- [ ] **Step 6.1: Crear branch**

```bash
cd /Users/alfonsogarre/Workspace/jota-transcriber
git checkout -b feat/docs-org-context
```

- [ ] **Step 6.2: Añadir badge de estado y contexto de org al inicio**

Al inicio de `README.md`, añadir (antes del primer badge existente):

```markdown
![Status: Maintained](https://img.shields.io/badge/status-Maintained-2ea44f)

> **Role in Jota ecosystem:** STT streaming microservice for the Jota gateway. Receives PCM Float32 audio over WebSocket and emits partial + final transcriptions. Receives `language` and `vad_thold` from the gateway's per-client `ClientConfig`.
>
> Part of [Jota-project](https://github.com/Jota-project). See [`ARCHITECTURE.md`](https://github.com/Jota-project/.github/blob/main/ARCHITECTURE.md) for the full system map.

---
```

- [ ] **Step 6.3: Anotar la sección de auth con la migración planeada**

Buscar la sección **"With external auth API"** (en Quick Start, sección "Run"). Justo después del bloque del segundo ejemplo, añadir:

```markdown
> **Note on auth:** `AUTH_API_URL` was originally designed to point to `jota-db`. The recommended path forward is per-service `AUTH_TOKEN` (static), with `AUTH_API_URL` kept only for setups that still centralize auth via `jota-db`. The deprecation of `jota-db` as default auth backend is tracked in [`Jota-project/jota-gateway` issues](https://github.com/Jota-project/jota-gateway/issues).
```

- [ ] **Step 6.4: Añadir sección "Status & roadmap" al final**

```markdown
## Status & roadmap

- **Status:** Maintained. Recent work (2026-05) focused on stability fixes — `ModelCache` RAII, `AudioDecoder` robustness, `InferenceLimiter` TOCTOU.
- **Known issue:** race condition between `flushLoop` and `handleEnd` may emit duplicate `is_final` transcriptions. Fix is straightforward; see issue #27.
- **Auth migration:** the default auth path will move from `jota-db` external API to per-service `AUTH_TOKEN` static. `AUTH_API_URL` will remain as an option.
```

- [ ] **Step 6.5: Commit**

```bash
cd /Users/alfonsogarre/Workspace/jota-transcriber
git add README.md
git commit -m "docs: add org context and annotate auth migration direction

Refs: docs/superpowers/specs/2026-07-01-jota-docs-alignment-design.md

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 6.6: Push y abrir PR**

```bash
cd /Users/alfonsogarre/Workspace/jota-transcriber
git push -u origin feat/docs-org-context
gh pr create --base main --head feat/docs-org-context \
  --title "docs(transcriber): add org context and annotate auth migration" \
  --body "Edición quirúrgica: añade badge Maintained, contexto de org, nota sobre la migración de auth planeada (jota-db → AUTH_TOKEN per-service), y sección Status & roadmap. Sin cambios funcionales."
```

---

### Task 7: Crear issue en `jota-gateway` sobre deprecar jota-db como auth backend

**Files:** ninguno (es una issue de GitHub).

- [ ] **Step 7.1: Verificar el clon local de `jota-gateway`**

```bash
ls /Users/alfonsogarre/Workspace/jota-gateway
cd /Users/alfonsogarre/Workspace/jota-gateway
git remote -v
git branch --show-current
```

Expected: remote `https://github.com/Jota-project/jota-gateway.git`, branch `main`, working tree clean. (Ya confirmado por el usuario; no requiere clone.)

- [ ] **Step 7.2: Crear la issue**

```bash
gh issue create --repo Jota-project/jota-gateway \
  --title "chore: deprecate jota-db as auth backend for transcriber and speaker" \
  --label "enhancement,cleanup,priority: low" \
  --body "$(cat <<'EOF'
## Context

`jota-gateway` v1.9.0 (PR #67) replaced `jota-db` as the source of truth for client identity and configuration with a local SQLite database. However, `jota-transcriber` and `jota-speaker` still depend on `jota-db` to validate tokens via an external auth API.

## Proposal

Migrate transcriber and speaker to a simpler auth model:

- Each service has its own static `AUTH_TOKEN` (per-service) as the default.
- `AUTH_API_URL` remains as an option for setups that still centralize auth via `jota-db`.
- Remove the default dependency in upcoming PRs.

## Acceptance criteria

- Document the new auth model in each repo's README and how to migrate existing setups.
- Mark `jota-db` external auth API as **optional** in the READMEs of `jota-transcriber` and `jota-speaker` (already in deprecation by default — see spec for context).
- Add a release note in `TASKS.md` of the org root.

## Related

- PR #67 (gateway SQLite)
- Spec: [`Jota-project/.github/docs/superpowers/specs/2026-07-01-jota-docs-alignment-design.md` §6.1](https://github.com/Jota-project/.github/blob/main/docs/superpowers/specs/2026-07-01-jota-docs-alignment-design.md)
EOF
)"
```

Expected: URL de la issue devuelta por `gh`. Anotarla.

- [ ] **Step 7.3: Anotar la URL en `TASKS.md` raíz**

Editar `/Users/alfonsogarre/Workspace/jota_.github/TASKS.md`, en la sección "#### (nueva) — Deprecar jota-db como auth backend en transcriber/speaker", añadir al final:

```markdown

Issue abierta: <pegar URL devuelta por gh issue create>
```

Hacer commit del cambio:

```bash
cd /Users/alfonsogarre/Workspace/jota_.github
git add TASKS.md
git commit -m "docs(tasks): link to new issue on deprecating jota-db as auth backend

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
git push origin feat/docs-alignment-root
```

(Ten en cuenta: este commit va al mismo branch `feat/docs-alignment-root` que el PR-1; aparecerá como push adicional al mismo PR. Alternativa: commit aparte en `main` directo si prefieres no tocar el PR-1.)

---

### Task 8: Post-transfer — verificar `jota-voice` y `jota-display`

**Pre-requisito (del usuario, no del agente):** el usuario debe haber transferido `SitoSt/jota-voice` y `SitoSt/jota-display` a `Jota-project/*` desde la UI de GitHub.

- [ ] **Step 8.1: Esperar confirmación del usuario**

Confirmar con el usuario que los transfers están hechos. Si no, parar este task.

- [ ] **Step 8.2: Cambiar el remote de los clones locales**

```bash
cd /Users/alfonsogarre/Workspace/jota-voice
git remote set-url origin https://github.com/Jota-project/jota-voice.git
git fetch origin
git branch --set-upstream-to=origin/main main 2>/dev/null || git checkout -b main origin/main

cd /Users/alfonsogarre/Workspace/jota-display
git remote set-url origin https://github.com/Jota-project/jota-display.git
git fetch origin
git branch --set-upstream-to=origin/main main 2>/dev/null || git checkout -b main origin/main
```

- [ ] **Step 8.3: Buscar referencias a la URL antigua en cada repo**

```bash
cd /Users/alfonsogarre/Workspace/jota-voice
grep -rn "SitoSt/jota-voice" --include="*.md" --include="*.yml" --include="*.yaml" --include="*.sh" --include="*.py" .
cd /Users/alfonsogarre/Workspace/jota-display
grep -rn "SitoSt/jota-display" --include="*.md" --include="*.yml" --include="*.yaml" --include="*.sh" --include="*.py" .
```

- [ ] **Step 8.4: Reemplazar las referencias encontradas**

Si el grep devuelve resultados, usar `sed` para reemplazar:

```bash
cd /Users/alfonsogarre/Workspace/jota-voice
grep -rl "SitoSt/jota-voice" --include="*.md" --include="*.yml" --include="*.yaml" . | xargs sed -i '' 's|SitoSt/jota-voice|Jota-project/jota-voice|g'

cd /Users/alfonsogarre/Workspace/jota-display
grep -rl "SitoSt/jota-display" --include="*.md" --include="*.yml" --include="*.yaml" . | xargs sed -i '' 's|SitoSt/jota-display|Jota-project/jota-display|g'
```

Re-verificar con el mismo grep; debería devolver cero resultados.

- [ ] **Step 8.5: Commit + PR en cada repo (solo si hubo cambios)**

`jota-voice`:
```bash
cd /Users/alfonsogarre/Workspace/jota-voice
git checkout -b chore/transfer-urls
git add -u
git diff --cached --quiet || git commit -m "chore: update URLs after transfer to Jota-project org

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
git push -u origin chore/transfer-urls
gh pr create --base main --head chore/transfer-urls \
  --title "chore(voice): update internal references to Jota-project URLs" \
  --body "Reemplaza referencias a SitoSt/jota-voice por Jota-project/jota-voice tras el transfer a la org."
```

`jota-display` (mismo flujo):
```bash
cd /Users/alfonsogarre/Workspace/jota-display
git checkout -b chore/transfer-urls
git add -u
git diff --cached --quiet || git commit -m "chore: update URLs after transfer to Jota-project org

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
git push -u origin chore/transfer-urls
gh pr create --base main --head chore/transfer-urls \
  --title "chore(display): update internal references to Jota-project URLs" \
  --body "Reemplaza referencias a SitoSt/jota-display por Jota-project/jota-display tras el transfer a la org."
```

Si no hubo cambios, omitir el PR y documentar en el log.

---

### Task 9: Verificación final de criterios de done

**Files:** ninguno — esta tarea solo verifica.

- [ ] **Step 9.1: Verificar que el README raíz se ve en GitHub**

```bash
gh api repos/Jota-project/.github/contents/README.md --jq '.download_url'
```

Expected: URL al README en la raíz del repo (no bajo `profile/`).

- [ ] **Step 9.2: Verificar que la carpeta `profile/` ya no existe**

```bash
gh api repos/Jota-project/.github/contents/profile 2>&1 | head -5
```

Expected: error 404 / not found.

- [ ] **Step 9.3: Buscar referencias muertas en toda la doc de la org**

```bash
for repo in jota-gateway jota-orchestrator jota-inference jota-db jota-transcriber jota-speaker; do
  echo "=== $repo ==="
  gh api "repos/Jota-project/$repo/contents/README.md" --jq '.download_url' | xargs curl -s | grep -E "SitoSt/jota-voice|SitoSt/jota-display|profile/README" || echo "  (clean)"
done
```

Expected: ningún repo (excepto los transferidos) menciona los personales o `profile/README`.

- [ ] **Step 9.4: Verificar badges de estado**

Para cada repo, comprobar que el README contiene `Maintained`, `Alternative`, o `Deprecated` según lo esperado:

```bash
for repo in jota-speaker jota-transcriber; do
  echo "=== $repo (esperado: Maintained) ==="
  gh api "repos/Jota-project/$repo/contents/README.md" --jq '.download_url' | xargs curl -s | grep -E "status-Maintained|status-Alternative|status-Deprecated" || echo "  ⚠️ sin badge"
done

for repo in jota-orchestrator jota-inference; do
  echo "=== $repo (esperado: Alternative) ==="
  gh api "repos/Jota-project/$repo/contents/README.md" --jq '.download_url' | xargs curl -s | grep -E "status-Maintained|status-Alternative|status-Deprecated" || echo "  ⚠️ sin badge"
done

echo "=== jota-db (esperado: Deprecated) ==="
gh api "repos/Jota-project/jota-db/contents/README.md" --jq '.download_url' | xargs curl -s | grep -E "status-Maintained|status-Alternative|status-Deprecated" || echo "  ⚠️ sin badge"
```

Expected: cada repo tiene el badge esperado.

- [ ] **Step 9.5: Verificar la issue creada en Task 7**

```bash
gh issue list --repo Jota-project/jota-gateway --state open --label cleanup --json number,title | grep -i "deprecate jota-db"
```

Expected: la issue aparece listada.

- [ ] **Step 9.6: Resumen al usuario**

Reportar al usuario:
- Cuántos PRs abiertos (esperado: 6 si todo OK, 7-8 si transfers requirieron cambios).
- Cualquier bloqueo o desviación respecto al spec.
- PRs pendientes de revisión/merge.

---

## Self-Review (checks ya aplicados durante la escritura)

- **Cobertura del spec:** §3 (arquitectura destino) → Task 1 (ARCHITECTURE.md, README raíz), §4 (estructura .github) → Task 1, §5 (cambios por repo) → Tasks 1-6, §6 (issues nuevas) → Task 7, §7 (transfers) → Task 8, §10 (done) → Task 9.
- **Placeholders:** ningún "TBD" / "TODO" en los steps. Cada bloque de código está completo y listo para copy-paste.
- **Consistencia de tipos/nombres:** "OpenClaw" usado consistentemente; "jota-db" / "JotaDB" usados según el contexto (en docs externos, "jota-db"; en código de `jota-inference` que mencionaba `JotaDB` literal, se conserva).
- **Rama base:** `jota-inference` usa `master`, los demás `main`. Documentado en Global Constraints y reiterado en Task 4.
- **Remote discrepancy:** `JotaDB` clon local apunta a `SitoSt/JotaDB`. Documentado en Task 5.1 con paso explícito de redirección antes de pushear.
- **Tasks 7.3 decisión:** he hecho que el commit del link a la issue vaya al mismo branch del PR-1 (alternativa marcada en el propio step para quien prefiera commit aparte en main).

## Execution Handoff

**Plan completo y guardado en `docs/superpowers/plans/2026-07-01-jota-docs-alignment.md`.**

Dos opciones de ejecución:

1. **Subagent-Driven (recomendado)** — Despacho un subagente fresco por task, reviso entre tasks, iteración rápida.
2. **Inline Execution** — Ejecuto los tasks en esta sesión con `executing-plans`, batch con checkpoints.

**¿Qué opción prefieres?** (Nota: el Task 8 requiere tu acción previa — el transfer de los repos personales a la org — antes de poder ejecutarse.)