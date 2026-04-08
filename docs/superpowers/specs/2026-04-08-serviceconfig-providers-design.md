# Spec: ServiceConfig + InferenceProvider

> Fecha: 2026-04-08  
> Estado: Aprobado — pendiente de implementación  
> Afecta a: `jota-db`, `jota-orchestrator`, `jota-gateway`

---

## Contexto

Jota es un sistema **single-user, multi-device**: un único propietario con múltiples clientes (dispositivos). Los "clientes" son altavoces, pantallas, dispositivos del jardín, etc. — no usuarios independientes.

Hoy, cada microservicio se configura exclusivamente vía `.env`. Esto impide modificar parámetros operativos en caliente, dificulta la administración centralizada y no escala cuando el usuario quiere añadir nuevos providers de IA o ajustar parámetros de servicios sin reiniciar contenedores.

Este diseño establece dos nuevas capas en `jota-db`:

1. **`ServiceConfig`** — configuración operativa genérica por servicio (reemplaza `.env` progresivamente)
2. **`InferenceProvider`** — entidad estructurada para backends de IA (local, OpenAI, Anthropic, custom)

---

## Diseño

### 1. Tabla `service_config`

Almacena parámetros de operación de cada microservicio como pares clave-valor JSON.

```
service_config
──────────────────────────────────────────────
service      TEXT NOT NULL   -- "transcriber" | "orchestrator" | "speaker" | "gateway" | "inference"
key          TEXT NOT NULL   -- nombre del parámetro, ej. "model", "audio.chunk_ms"
value        JSON NOT NULL   -- cualquier tipo: string, number, bool, object
description  TEXT            -- descripción legible para admin
updated_at   DATETIME

PRIMARY KEY (service, key)
```

**Ejemplos de registros:**

| service | key | value |
|---|---|---|
| `transcriber` | `model` | `"whisper-large-v3"` |
| `transcriber` | `audio.chunk_ms` | `200` |
| `orchestrator` | `default_provider_id` | `"uuid-del-provider"` |
| `orchestrator` | `fallback_provider_id` | `null` |
| `speaker` | `model` | `"kokoro-v1"` |
| `speaker` | `default_voice` | `"af_heart"` |
| `speaker` | `default_speed` | `1.0` |

---

### 2. Tabla `inference_provider`

Entidad estructurada para cada backend de inferencia disponible en el sistema.

```
inference_provider
──────────────────────────────────────────────
id                UUID PRIMARY KEY
name              TEXT NOT NULL        -- display name, ej. "OpenAI GPT-4o"
type              TEXT NOT NULL        -- "local" | "openai" | "anthropic" | "custom"
base_url          TEXT                 -- requerido para type=local y type=custom
api_key           TEXT                 -- para type=openai y type=anthropic (cifrado en producción)
default_model_id  TEXT                 -- ej. "gpt-4o", "claude-opus-4-6", "llama-3.2-3b"
is_active         BOOLEAN DEFAULT TRUE
extra_config      JSON                 -- config adicional específica del provider
created_at        DATETIME
updated_at        DATETIME
```

**Tipos de provider:**

| type | base_url | api_key | Descripción |
|---|---|---|---|
| `local` | `ws://jota-inference:8002` | — | jota-inference (llama.cpp), ya existente |
| `openai` | — (usa SDK oficial) | `sk-...` | OpenAI API |
| `anthropic` | — (usa SDK oficial) | `sk-ant-...` | Anthropic API |
| `custom` | `http://mi-vps:8080` | opcional | Endpoint compatible OpenAI en servidor propio |

---

### 3. Seeding desde `.env`

Al arrancar jota-db, si las tablas están vacías, se pueblan desde variables de entorno. Esto garantiza que el sistema funcione desde el primer arranque sin configuración manual en la DB.

**Variables de entorno de seeding (jota-db):**

```env
# Provider local (siempre presente)
SEED_LOCAL_PROVIDER_URL=ws://jota-inference:8002
SEED_LOCAL_MODEL_ID=llama-3.2-3b

# Providers externos (opcionales)
SEED_OPENAI_API_KEY=sk-...
SEED_OPENAI_DEFAULT_MODEL=gpt-4o

SEED_ANTHROPIC_API_KEY=sk-ant-...
SEED_ANTHROPIC_DEFAULT_MODEL=claude-sonnet-4-6

# ServiceConfig por servicio
SEED_TRANSCRIBER_MODEL=whisper-large-v3
SEED_TRANSCRIBER_AUDIO_CHUNK_MS=200
SEED_SPEAKER_MODEL=kokoro-v1
SEED_ORCHESTRATOR_DEFAULT_PROVIDER=local  # nombre o id del provider
```

El seeding se ejecuta **una sola vez** al arrancar si las tablas están vacías. Los cambios posteriores vía API persisten y tienen prioridad.

> **Nota de migración:** mientras jota-db no tenga persistencia estable, los datos se pierden en restart y se reseedan desde `.env`. Es el comportamiento esperado en esta fase.

---

### 4. Endpoints admin en jota-db

#### ServiceConfig

| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/admin/config` | Listar toda la config de servicios |
| `GET` | `/admin/config/{service}` | Config de un servicio específico |
| `PUT` | `/admin/config/{service}/{key}` | Crear o actualizar un valor |
| `DELETE` | `/admin/config/{service}/{key}` | Eliminar un valor (vuelve al default) |
| `POST` | `/admin/config/{service}/reset` | Restaurar defaults del servicio desde `.env` |

#### InferenceProvider

| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/admin/providers` | Listar providers (activos e inactivos) |
| `POST` | `/admin/providers` | Crear nuevo provider |
| `GET` | `/admin/providers/{id}` | Detalle de un provider |
| `PUT` | `/admin/providers/{id}` | Actualizar (API key, activar/desactivar, modelo default) |
| `DELETE` | `/admin/providers/{id}` | Desactivar provider (soft delete) |

**Auth:** misma clave `X-API-Key` de admin que el resto de endpoints de jota-db.

---

### 5. Exposición en jota-gateway

El gateway proxea los endpoints admin bajo `/api/admin/*`:

| Gateway | jota-db |
|---|---|
| `GET /api/admin/config` | `GET /admin/config` |
| `GET /api/admin/config/{service}` | `GET /admin/config/{service}` |
| `PUT /api/admin/config/{service}/{key}` | `PUT /admin/config/{service}/{key}` |
| `GET /api/admin/providers` | `GET /admin/providers` |
| `POST /api/admin/providers` | `POST /admin/providers` |
| `PUT /api/admin/providers/{id}` | `PUT /admin/providers/{id}` |

Auth: `X-API-Key` requerido (igual que el resto de la API).

---

### 6. Impacto en jota-orchestrator (fase siguiente)

Una vez jota-db implemente lo anterior, el orchestrator añade una capa de abstracción `LLMProvider`:

```
LLMProvider (interfaz)
├── LocalProvider      → WebSocket a jota-inference (comportamiento actual)
├── OpenAIProvider     → openai Python SDK
├── AnthropicProvider  → anthropic Python SDK
└── CustomHTTPProvider → endpoint compatible OpenAI
```

Al arrancar, el orchestrator llama a `GET /admin/config/orchestrator` para leer `default_provider_id`, y a `GET /admin/providers` para cargar los providers activos. Cachea con TTL largo (ej. 5 min) dado que este config cambia raramente.

La selección de provider por request sigue este orden de precedencia:
1. Override en `ClientConfig.preferred_model_id` (por dispositivo)
2. `orchestrator.default_provider_id` (default del sistema)

---

### 7. Orden de implementación recomendado

1. **jota-db:** modelos `ServiceConfig` + `InferenceProvider` + migraciones
2. **jota-db:** lógica de seeding desde `.env` al startup
3. **jota-db:** endpoints `/admin/config/*` y `/admin/providers/*`
4. **jota-gateway:** proxy de endpoints admin
5. **jota-orchestrator:** capa `LLMProvider` + lectura de config desde jota-db

---

### 8. Lo que NO cambia en esta fase

- `ClientConfig` (configuración por dispositivo) — no se toca en esta fase
- `jota-inference` — permanece sin cambios, es un backend más
- Protocolo WebSocket de voz — no se afecta
- Auth existente — mismos mecanismos

---

## Referencias

- Arquitectura actual: `ARCHITECTURE.md`
- Tareas pendientes: `TASKS.md`
- Repos: [jota-db](https://github.com/jota-project/jota-db) · [jota-orchestrator](https://github.com/jota-project/jota-orchestrator) · [jota-gateway](https://github.com/jota-project/jota-gateway)
