# Jota — Alineación de documentación y arquitectura tras la modularización

> Spec de diseño · 2026-07-01 · Estado: **borrador pendiente de aprobación**

---

## 1. Resumen ejecutivo

El ecosistema Jota ha cambiado sustancialmente en los últimos 3 meses, pero la documentación raíz de la organización (`Jota-project/.github`) está congelada en abril 2026 y los READMEs de varios repos están desalineados con el código real. Este spec describe cómo alinear **toda** la documentación de la organización con la arquitectura vigente: una capa de gateway modular con SQLite local, el orquestador open source **OpenClaw** como camino LLM recomendado, y dos clientes nuevos (jota-voice, jota-display) que pasan de la cuenta personal del autor a la organización.

El resultado es que cualquier visitante nuevo de la org entiende en menos de 2 minutos qué hace Jota hoy, qué servicios están mantenidos, cuáles son alternativa y cuálesDeprecated, y dónde profundizar.

---

## 2. Estado actual (snapshots verificados, jul 2026)

### 2.1 Repos de la organización `Jota-project`

| Repo | Lenguaje | Último push | Commits recientes relevantes |
|---|---|---|---|
| `jota-gateway` | Python/FastAPI | 2026-07-01 | v1.9.0 — reemplazo de jota-db por SQLite local (PR #67); API redesign (#66); OpenClaw multiplexed (#64); push_enabled + max_silence_turns |
| `jota-orchestrator` | Python/FastAPI | 2026-04-05 | Última actividad útil: QuickRequest con system_prompt_extra (#18) |
| `jota-inference` | C++/llama.cpp | 2026-03-16 | Sin actividad desde marzo; release 1.1.0 |
| `jota-db` | Python/FastAPI/SQLModel | 2026-04-10 | ServiceConfig / InferenceProvider / AdminUser; routers `/internal/` y `/admin/` |
| `jota-transcriber` | C++17/whisper.cpp | 2026-05-15 | Deps bump + 1.1.x fixes (buffer_mutex_, InferenceLimiter, ModelCache, AudioDecoder) |
| `jota-speaker` | Python/Kokoro | 2026-05-26 | Wyoming protocol para Home Assistant (puerto 20424); voz `ef_dora` |
| `.github` | — | 2026-07-01 | ARCHITECTURE.md congelado en abril; profile/README.md previo |

### 2.2 Repos personales del autor (SitoSt)

Activos y relevantes para este spec:

| Repo | Rol | Acción |
|---|---|---|
| `SitoSt/jota-voice` | Cliente Termux/Android; WS streaming directo al gateway | **Transferir** a `Jota-project/jota-voice` |
| `SitoSt/jota-display` | UI kiosk Vue 3 + SSE en el dispositivo | **Transferir** a `Jota-project/jota-display` |
| `SitoSt/jota-wake-trainer` | CLI openWakeWord trainer | Mantener personal; mencionar brevemente |
| `SitoSt/jota-dashboard` | Dashboard web | Mantener personal; mencionar brevemente |
| `SitoSt/jota-float`, `jota-swift`, `JotaDesktop`, `JotaClient`, etc. | Otros experimentos | No se referencian en doc de la org |

### 2.3 Lo que la documentación actual dice (incorrecto o desactualizado)

- `ARCHITECTURE.md` raíz menciona jota-orchestrator + jota-inference + jota-db como camino activo.
- `profile/README.md` lista 6 servicios con badge `Stable` para todos, incluido jota-inference.
- El jota-db tiene un README con ejemplos de `curl` para Tasks/Events que parecen ser de otro proyecto previo y no encajan con su rol real (identidad/clientes/auth).
- Los READMEs de jota-speaker, jota-orchestrator, jota-inference están desconectados de la org (sin enlaces cruzados, sin contexto de ecosistema).
- jota-transcriber sigue considerando jota-db como fuente de auth canónica (vía `AUTH_API_URL`).

---

## 3. Arquitectura destino

### 3.1 Servicios por categoría

**Maintained** — Camino recomendado; desarrollo activo.

| Repo | Rol | Notas |
|---|---|---|
| `jota-gateway` | BFF único (puerto 8004). 4 superficies: WS `/ws/stream`, Admin REST `/admin/*`, OpenAI-compat `/v1/*`, Health `/healthz` `/ready`. SQLite local para identidad/clientes/config. | Recién reescrito (v1.9.0). No tocar el README salvo revisión sin cambios. |
| `jota-transcriber` | Whisper C++17 streaming (puerto 9000). WebSocket JSON. | Activo. Editar README para anotar la migración prevista de auth (ver §6). |
| `jota-speaker` | Kokoro TTS (puerto 8005) + Wyoming protocol (puerto 20424) para Home Assistant. Voz `ef_dora`. | Activo. Reencuadrar README con contexto de org. |
| `jota-voice` | Cliente Termux/Android, streaming WS directo al gateway. | **Transferir** desde `SitoSt/`. |
| `jota-display` | UI kiosk Vue 3 + SSE en dispositivo. | **Transferir** desde `SitoSt/`. |

**Alternative** — Camino 100% local legacy; menos desarrollo futuro. Mantener por compatibilidad pero ya no es el recomendado.

| Repo | Rol | Notas |
|---|---|---|
| `jota-orchestrator` | Orquestador LLM Python (puerto 8000). Conectable desde gateway. | Recomendación: migrar a `llama.cpp + OpenClaw`. Reencuadrar README. |
| `jota-inference` | Motor llama.cpp C++ (puerto 3000). | Sin commits desde marzo. Recomendación: igual, migrar a `llama.cpp` standalone + OpenClaw. Reencuadrar README. |

**Deprecated** — Mantener por compatibilidad pero migrar fuera.

| Repo | Rol | Notas |
|---|---|---|
| `jota-db` | Identidad/clientes/auth Python (puerto 8001). Gateway ya no lo usa como fuente de verdad. Sigue siendo opción de "central auth API" para setups que aún lo quieran. | Marcar como Deprecated. Mover sección Tasks/Events del README a `docs/legacy-tasks-api.md`. |

### 3.2 Diagrama lógico

```
Cliente físico (ESP32 / App / Web / Home Assistant / Termux)
        │
        ▼
  ┌────────────────────────────────────────────────────┐
  │                  jota-gateway (BFF)                │
  │  SQLite local · 4 superficies · WS por sesión      │
  └──┬─────────────┬─────────────┬─────────────┬──────┘
     │             │             │             │
     ▼             ▼             ▼             ▼
  OpenClaw    jota-transcriber  jota-speaker  jota-db (solo
  (LLM +      (Whisper C++)     (Kokoro +     como auth
   tools)                        Wyoming)      externa opcional)
                                       │
                                       ▼
                              Wyoming TCP 20424 → Home Assistant

Clientes companion:
  jota-voice  ──┐
  jota-display ─┴──► ambos corren en el mismo dispositivo (Android/Termux)
                       y se conectan al gateway por LAN
```

### 3.3 Clientes que viven en la org vs personales

Documentados en la org como "Clients":
- `jota-voice` (transferido)
- `jota-display` (transferido)

Mención breve con link, fuera de la tabla principal:
- `SitoSt/jota-wake-trainer`
- `SitoSt/jota-dashboard`

Resto de `SitoSt/*` no se menciona en la doc de la org.

---

## 4. Estructura final del repo `Jota-project/.github`

### 4.1 Layout

```
Jota-project/.github/
├── README.md                                ← sale del repo y se muestra en la página de la org
├── ARCHITECTURE.md                          ← reescrito (alta densidad, legacy preservado)
├── TASKS.md                                 ← actualizado con issues activas + nuevas
├── CONTRIBUTING.md                          ← workflows actuales + nota de estado
├── docs/
│   └── superpowers/
│       └── specs/
│           └── 2026-07-01-jota-docs-alignment-design.md   ← este spec
```

Se **borra** la carpeta `profile/`. Razón: GitHub ya no requiere el prefijo `profile/` para detectar el README especial de la org; lee directamente `README.md` en la raíz del repo `.github`. Migrar contenido y borrar carpeta.

### 4.2 Contenido de `README.md` (raíz)

- **Tagline + descripción de una frase** sobre Jota.
- **Diagrama de arquitectura simplificado** mostrando solo los servicios Maintained + Alternative en gris punteado.
- **Tabla "Services"** con badge de estado (`Maintained` / `Alternative` / `Deprecated`) y link a cada repo.
- **Sección "Clients"** con `jota-voice` y `jota-display` (tabla con descripción corta) + nota breve sobre los que siguen en `SitoSt/*` (un párrafo con lista de links).
- **Sección "Quick start"** — un párrafo con `git clone` de los 4-5 repos Maintained y enlace a `ARCHITECTURE.md`.
- **Sección "Status & roadmap"** — 2-3 viñetas sobre la dirección actual (OpenClaw + SQLite en gateway, modularización, foco en clientes).

### 4.3 Contenido de `ARCHITECTURE.md`

Estructura de secciones (orden pensado para que un nuevo contributor lo lea de arriba abajo):

1. **Visión general** — qué es Jota y para quién.
2. **Mapa de servicios** — diagrama + tabla con estado y rol de cada uno.
3. **Gateway — API pública** — las 4 superficies (WS, Admin REST, OpenAI-compat, Health) con tablas de endpoints.
4. **Flujo de identidad** — diagrama de handshake y propagación de ClientConfig desde el gateway a los servicios.
5. **ClientConfig** — tabla de campos tal como están hoy en `ClientRecord` del gateway (SQLite local), con defaults actuales.
6. **Protocolos de los microservicios**:
   - `jota-transcriber` (handshake, mensajes, auth).
   - `jota-speaker` (WebSocket + Wyoming).
   - OpenClaw (resumen, sin replicar docs upstream).
   - `jota-orchestrator` (solo referencia; marcar como Alternative).
7. **Caché en el gateway** — TTLs y maxsizes reales.
8. **Clientes** — `jota-voice` y `jota-display` con una subsección cada uno.
9. **Issues activas** — tabla resumen con las que están abiertas en cada repo.
10. **Estructura de ficheros clave** — por repo, qué archivo tiene qué responsabilidad.
11. **Legacy / Deprecated** — sección final con explicación de por qué jota-inference, jota-orchestrator y jota-db están en su estado actual y la recomendación de migrar.

### 4.4 Contenido de `TASKS.md`

- **Estado del sistema** — 4-5 viñetas sobre el estado actual.
- **Issues activas por repo** — tabla o secciones por repo con las issues abiertas.
- **Issues nuevas a crear durante este trabajo** (ver §6).
- **Changelog breve** — issues cerradas recientemente con una línea cada una (referencia, no detalle).

### 4.5 Contenido de `CONTRIBUTING.md`

- Workflow de Docker (existente, válido).
- Linting: `ruff` para Python, Clang-Format para C++.
- Pull request process (existente).
- **Sección nueva**: "Estado de los repos" — link al README raíz y nota de que `jota-orchestrator`, `jota-inference` y `jota-db` están en modo Alternative/Deprecated y los PRs van contra los Maintained primero.

---

## 5. Cambios por repo individual

### 5.1 `jota-gateway`

**Acción: solo revisión, sin cambios en README.**

Razón: reescrito hoy (v1.9.0, 1 jul 2026) y CLAUDE.md actualizado en la misma serie de PRs. Riesgo de romper algo que ya está bien.

Revisión que hago (sin commit):
- Verificar que el README menciona correctamente OpenClaw, las 4 superficies, y que la sección de env vars está sincronizada con CLAUDE.md.
- Verificar que no diga "fuente de verdad jota-db" en ningún sitio (debería decir SQLite local).
- Si encuentro algo objetivamente mal, lo marco en el spec como follow-up pero no edito sin tu OK explícito.

### 5.2 `jota-transcriber`

**Acción: edición quirúrgica del README.**

Añadir al README:
- Badge de estado `Maintained`.
- Sección "Role in Jota ecosystem" (2-3 frases): es el STT del gateway, recibe audio PCM Float32, devuelve transcripciones parciales y finales por WebSocket.
- Una nota explícita en la sección de auth: *"jota-db es una opción de external auth API; el camino recomendado a corto plazo es `AUTH_TOKEN` estático (per-service). Issue abierta: ver [link a issue §6.1]."*
- Link a `Jota-project/.github/blob/main/ARCHITECTURE.md`.

### 5.3 `jota-speaker`

**Acción: reescritura media del README.**

Razón: README técnico bueno, pero sin contexto de org.

Estructura nueva:
- Badge `Maintained`.
- Sección "Role in Jota ecosystem" (cómo encaja con gateway + transcriber + OpenClaw).
- Link al repo raíz.
- Quick start (ya está, conservar).
- WebSocket protocol (ya está, conservar).
- **Sección nueva destacada**: "Wyoming protocol for Home Assistant" — el README actual la tiene al final; subirla y darle más peso, porque es el camino nativo de HA y es lo que más interesa al visitante nuevo.
- HTTP endpoints (ya está, conservar).
- Configuration (ya está, conservar).
- Status & roadmap (nueva): 2-4 viñetas sobre commits recientes (Wyoming, ef_dora) y dirección.

### 5.4 `jota-orchestrator`

**Acción: reescritura media del README.**

Razón: detallado pero obsoleto (menciona `JOTA_DB_URL`, `Inference Center`, `JotaDB` como fuente de verdad).

Cambios:
- Badge `Alternative`.
- Banner al inicio: *"Este repo está en modo Alternative. El camino recomendado es OpenClaw (ver [link]). Mantenemos el código por compatibilidad con setups 100% locales."*
- Reescribir la sección de variables de entorno para reflejar que `JOTA_DB_URL` ya no se usa para conversaciones (ahora gateway → orchestrator vía `x-client-key` + `x-client-id` y la persistencia es local en gateway).
- Reducir/reescribir la sección de Tools para enfocarse solo en lo que sigue funcionando (Tavily, MCP, parsing de tool calls). Quitar claims sobre JotaDB.
- Mantener la sección de Testing (sigue válida).

### 5.5 `jota-inference`

**Acción: reescritura fuerte del README.**

Razón: el README actual es 100% sobre el binario `InferenceCore`, sin contexto de org, sin mención de Alternative, sin links.

Cambios:
- Badge `Alternative`.
- Banner: *"Este repo está en modo Alternative. Sin commits desde marzo 2026. Recomendamos usar `llama.cpp` standalone con OpenClaw. Ver [link al profile README]."*
- Quick start (ya está, conservar pero añadir nota: "puedes seguir usando este binario con jota-orchestrator si lo necesitas").
- WebSocket API (ya está, conservar).
- Auth (ya está, conservar).
- Eliminar referencias a "JotaDB" como central auth (sigue válido mencionarlo, pero como una opción entre varias, no como la única).
- Añadir sección "Role in Jota ecosystem" y link al repo raíz.

### 5.6 `jota-db`

**Acción: banner + reorganización.**

Razón: el README actual empieza con ejemplos de `curl` para Tasks/Events que parecen ser de otro proyecto previo y no encajan con el `jota-db` real.

Cambios:
- Badge `Deprecated`.
- Banner: *"jota-db ya no es la fuente de verdad de identidad/clientes. El gateway usa SQLite local. Este repo se mantiene como opción para centralizar auth de servicios en setups legacy."*
- Mover la sección "Tasks / Events" del README actual a `docs/legacy-tasks-api.md` con un banner indicando que esa parte está en desuso.
- El README reescrito empieza con el rol real: routers `/internal/` y `/admin/`, ServiceConfig, InferenceProvider, AdminUser, Client/ClientConfig.
- Link al repo raíz.

### 5.7 `jota-voice` y `jota-display`

**Acción: solo confirmar post-transfer.**

Razón: el usuario transfiere desde la UI de GitHub; yo reviso post-transfer que:
- Las URLs internas que apunten a `SitoSt/...` se actualicen a `Jota-project/...`.
- Los badges de CI apunten al repo nuevo.
- El README siga vigente (ambos están bien documentados).
- La mención en el README raíz + ARCHITECTURE.md los apunte correctamente.

---

## 6. Issues nuevas a crear

### 6.1 En `Jota-project/jota-gateway`

**Título:** `chore: deprecate jota-db as auth backend for transcriber and speaker`

**Cuerpo:**

> **Contexto**
>
> `jota-gateway` v1.9.0 (PR #67) reemplazó `jota-db` como fuente de verdad de identidad y configuración de clientes por una base SQLite local. Sin embargo, `jota-transcriber` y `jota-speaker` siguen dependiendo de `jota-db` para validar tokens vía external auth API.
>
> **Propuesta**
>
> Migrar transcriber y speaker a un modelo de auth más simple:
> - Cada servicio tiene su propio `AUTH_TOKEN` estático (per-service) como default.
> - `AUTH_API_URL` queda como opción para setups que aún quieran centralizar auth vía `jota-db`.
> - Eliminar la dependencia por defecto en próximos PRs.
>
> **Criterios de aceptación**
> - Documentar en cada repo el nuevo modelo de auth y cómo migrar setups existentes.
> - Marcar `jota-db` external auth API como ruta **opcional** en los README de transcriber y speaker (ya en desuso por defecto).
>
> **Relacionado**
> - PR #67 (gateway SQLite)
> - Ver también: `Jota-project/jota-db` marcado como Deprecated en la doc de la org.

**Labels:** `enhancement`, `cleanup`, `priority: low`.

### 6.2 No se crea issue nueva — referencia a #52

La issue #52 (`security: /v1/* endpoints exposed externally without auth`) ya está abierta en `jota-gateway`. Se documenta como follow-up en `TASKS.md` con nota: *"esperar decisión sobre el modelo de auth de `/v1/*` y reflejar en ARCHITECTURE.md cuando se cierre"*.

---

## 7. Cambios operativos sobre transfers

### 7.1 Procedimiento

1. El usuario transfiere `SitoSt/jota-voice` y `SitoSt/jota-display` a `Jota-project/*` desde la UI de GitHub (Settings → Danger Zone → Transfer). Historial de commits, issues y PRs se conserva.
2. Una vez transferidos, los clones locales se actualizan con `git remote set-url` a la nueva URL.
3. Búsqueda global de referencias a `SitoSt/jota-voice` y `SitoSt/jota-display` en toda la doc de la org; reemplazar por `Jota-project/...`.
4. Confirmar que badges de CI y links externos siguen funcionando.

### 7.2 Lo que NO se hace

- No se hace un clon-push forzado (perdería historial de issues).
- No se modifican los commits (transfer preserva el historial tal cual).
- No se mueven las issues abiertas (si las hay) — quedan donde están, solo cambia el namespace.

---

## 8. Riesgos y trade-offs

| Riesgo | Mitigación |
|---|---|
| Reescribir muchos READMEs a la vez introduce typos o cambios de tono no deseados | Hacerlo repo a repo, con diffs pequeños y revisables. Confirmar con el usuario antes de hacer commit en cada uno. |
| El transfer de GitHub puede romper badges CI si los workflows asumen paths específicos | Revisar workflows tras el transfer; actualizar paths si hace falta. |
| Los usuarios que tenían links a `SitoSt/jota-voice` o `SitoSt/jota-display` no se redirigen automáticamente | GitHub sí redirige (vanity redirect en el repo transferido), pero añadir nota en el README raíz sobre la nueva ubicación. |
| Editar `jota-orchestrator` README contradice su estado de congelado (no recibe commits desde abril) | Marcar explícitamente que el README documenta el código existente, no promesa de actividad. |
| Mover la sección Tasks/Events de jota-db a `legacy-tasks-api.md` puede romper links externos | Mantener un redirect en el README a la nueva ubicación. |

---

## 9. Plan de commits / PRs

Orden pensado para que cada cambio se pueda revisar aisladamente:

1. **PR 1 (en `.github`):** Reestructurar el repo raíz (mover `profile/README.md` → `README.md`, reescribir `ARCHITECTURE.md`, `TASKS.md`, `CONTRIBUTING.md`).
2. **PR 2 (en `jota-speaker`):** Reencuadrar README con contexto de org.
3. **PR 3 (en `jota-orchestrator`):** Reencuadrar README, marcar como Alternative.
4. **PR 4 (en `jota-inference`):** Reencuadrar README, marcar como Alternative.
5. **PR 5 (en `jota-db`):** Marcar como Deprecated, mover sección Tasks a legacy doc.
6. **PR 6 (en `jota-transcriber`):** Edición quirúrgica (badge + nota de auth migration).
7. **Issue 1 (en `jota-gateway`):** Crear issue sobre deprecación de jota-db como auth backend en transcriber/speaker.
8. **Tras transfer del usuario:** PRs en `jota-voice` y `jota-display` solo si hay URLs que actualizar.

El PR 1 no tiene dependencias. Los PRs 2-6 son independientes entre sí. Los PRs 7-8 son independientes y pueden ir en cualquier momento.

---

## 10. Criterios de "done"

- [ ] `Jota-project/.github/README.md` reemplaza a `profile/README.md` y se ve en la página principal de la org.
- [ ] `ARCHITECTURE.md` describe el sistema actual con la separación Maintained / Alternative / Deprecated.
- [ ] Los 7 READMEs de la org tienen badge de estado coherente.
- [ ] `jota-transcriber` y `jota-speaker` mencionan el issue de migración de auth.
- [ ] Las secciones obsoletas de `jota-db` (Tasks/Events) están en `docs/legacy-tasks-api.md` con banner.
- [ ] Las issues nuevas están creadas en sus repos correctos.
- [ ] Los transfers de `SitoSt/jota-voice` y `SitoSt/jota-display` están completados y los nuevos repos están enlazados desde la doc de la org.
- [ ] No hay referencias muertas a `SitoSt/jota-voice` o `SitoSt/jota-display` en la doc de la org (solo pueden quedar referencias a otros repos personales que sí siguen ahí).
- [ ] `TASKS.md` está sincronizado con las issues reales abiertas en GitHub.

---

## 11. Fuera de alcance

- Implementar los cambios descritos en las issues nuevas (cada issue genera su propio spec/plan si se decide trabajar en ello).
- Migrar código real de transcriber/speaker a keys propias (issue §6.1).
- Transferir `jota-wake-trainer`, `jota-dashboard` u otros personales a la org (decidido mantenerlos personales).
- Reescribir READMEs de los personales para alinearlos con la org.
- Cambios funcionales en el código de cualquier repo (solo docs).