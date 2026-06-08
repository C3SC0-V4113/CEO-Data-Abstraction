# Mirador - Propuesta de Arquitectura: Chatbot Ejecutivo (Text-to-SQL + RAG) Web SSR + MCP

**Mirador** es el nombre del producto: un chatbot ejecutivo que da una vista panoramica
del negocio combinando metricas (Text-to-SQL gobernado) y conocimiento documental (RAG).
Los servicios siguen el esquema `mirador-*` (`mirador-web`, `mirador-core`, `mirador-mcp`,
`mirador-ingestion`).

## Objetivo

Construir un sistema de consulta de datos asistido por IA para un CEO de una
empresa desarrolladora de software. El sistema debe aceptar lenguaje natural,
generar SQL seguro de solo lectura, devolver respuestas ejecutivas y renderizar
tablas, KPIs o graficas dinamicas dentro de una experiencia de chatbot.

Ademas de metricas, el chat debe responder preguntas **documentales no-metricas**
(vision, mision, a que se dedica la empresa, politicas, procesos) usando una **Capa de
Conocimiento (RAG)** sobre documentos heterogeneos (PDF, Word, presentaciones). Un mismo
prompt puede pedir metrica y conocimiento a la vez, y ambas deben volver en **una sola
respuesta**. El detalle de esa capa esta en
[rag-knowledge-layer.md](rag-knowledge-layer.md) y la decision en ADR-0008.

Debe existir acceso desde dos frentes:

- Web app con login y chatbot ejecutivo.
- Cliente MCP remoto compatible con Claude Desktop, Cursor/Codex u otros.

La web ya no es report-first ni dashboard-first. Para el MVP, las interfaces
propias son solo login y chatbot.

## Stack Propuesto

| Capa | Tecnologia | Despliegue Preferido |
| --- | --- | --- |
| Frontend web | Next.js SSR + shadcn/ui | Cloudflare Workers con OpenNext/Cloudflare |
| API Gateway web | Cloudflare (API Shield/WAF/Rate Limiting) | Borde Cloudflare frente al backend |
| API Gateway MCP | Cloudflare (API Shield/WAF/Rate Limiting) | Borde Cloudflare frente al servicio MCP |
| Backend API | Fastify + TypeScript | Railway recomendado para MVP |
| MCP remoto | Servicio MCP independiente + MCP TypeScript SDK | Servicio propio en Railway (no Fastify) |
| ORM | Prisma ORM | Backend Fastify |
| Base de datos | PostgreSQL | Railway Postgres recomendado para MVP |
| Vector store (RAG) | PostgreSQL + extension `pgvector` | Mismo Postgres de Railway |
| Ingesta RAG | Servicio TypeScript independiente (`mirador-ingestion`) | Railway |
| Object storage docs | Cloudflare R2 (archivos fuente RAG) | Cloudflare |
| Embeddings | `text-embedding-3-small` (default, configurable via `EMBEDDING_MODEL`) | Proveedor LLM (OpenAI) |
| Auth web | JWT + rol `CEO` | Fastify + cookies httpOnly o bearer interno |
| Secretos | Workers Secrets / Railway Variables | Segun capa |
| Reportes futuros | Queues/Workflows/R2 | Cloudflare opcional |

Nombres de repo por servicio: `mirador-web` (frontend), `mirador-core` (backend
Fastify), `mirador-mcp` (servicio MCP) y `mirador-ingestion` (ingesta RAG).

Para un MVP con menor riesgo operativo, el backend Fastify + Prisma y PostgreSQL
se desplegaran en Railway. Cloudflare Workers queda como opcion futura para el
backend si Prisma edge, Prisma Accelerate y el MCP SDK funcionan correctamente
en una prueba tecnica. El frontend SSR se mantiene en Cloudflare Workers.

## Componentes

### Next.js SSR + shadcn/ui

La web app sera una interfaz ejecutiva minimalista chat-first:

- Login obligatorio.
- Chatbot como unica vista autenticada.
- Preguntas sugeridas para CEO.
- Acciones rapidas dentro del chat.
- Composer con modos `responder`, `analizar`, `reporte_visual` y `plan`.
- Respuestas con tablas, KPIs, graficas y narrativa.
- Artefactos de reporte generados bajo demanda dentro del hilo.
- Mini chat contextual para editar graficas ya generadas.
- Filtros conversacionales para periodo, cliente, proyecto, area o metrica.
- Freshness, warnings y `trace_id` por respuesta.

Las server actions o route handlers de Next.js se usaran para UI/session cuando
convenga, pero las consultas de datos pasan por Fastify.

La web no consume MCP directamente. MCP es un canal externo para clientes como
Claude Desktop, Cursor/Codex u otros. La web usa APIs HTTP propias de Fastify
para evitar duplicar abstracciones y exponer una superficie innecesaria en el
browser.

Rutas SSR propuestas:

- `/login`
- `/chat`

`/` debe redirigir a `/chat` si existe sesion valida o a `/login` si no existe.

### API Gateway (Cloudflare)

El trafico entra por un borde de **dos gateways Cloudflare**, uno por servicio (ver
ADR-0007):

- **Web API Gateway**: frente al backend Fastify. Aplica politicas para el trafico
  interactivo del CEO.
- **MCP API Gateway**: frente al servicio MCP. Aplica politicas para el trafico
  programatico de clientes externos (cuotas por API key, limites mas estrictos).

Responsabilidades de ambos gateways: TLS, rate limiting, throttling, cuotas, WAF/IP
rules, limite de tamano de request, routing y chequeos coarse (presencia de JWT o API
key), mas observabilidad. Los gateways hacen control de trafico y auth coarse; la
autorizacion fina queda en los servicios (el backend valida firma JWT + rol; el servicio
MCP valida `MCP_API_KEY` + scope por tool).

La **API interna core** del backend (ver mas abajo) no se expone por el Web API Gateway;
es de uso interno entre servicios.

### Fastify + TypeScript

Fastify sera el backend principal:

- `POST /api/auth/login`
- `POST /api/auth/logout`
- `GET /api/auth/session`
- `POST /api/chat/messages`
- `GET /api/chat/conversations`
- `GET /api/chat/conversations/:conversation_id`
- `POST /api/chat/artifacts/:artifact_id/chart-edits`
- `GET /api/schema/catalog`
- `GET /api/query/history`
- `GET /api/health`
- `POST /internal/core/ask` (API interna; uso service-to-service, no publica)
- `GET /internal/core/schema-catalog` (API interna; uso service-to-service)

`POST /mcp` ya no vive en Fastify: se movio al **servicio MCP independiente**
(ver mas abajo y ADR-0007).

Fastify concentra autenticacion, JWT, autorizacion por rol, LLM Orchestrator,
Capa Semantica, validacion SQL, ejecucion read-only, auditoria y respuesta comun
en TypeScript. Ademas expone una **API interna core** (`/internal/core/*`) que
publica ese pipeline a llamadores internos de confianza (el servicio MCP) con auth
service-to-service (`CORE_SERVICE_TOKEN` y/o red privada de Railway); esta API no se
enruta por el Web API Gateway.

### Authentication and Authorization

El MVP tendra login obligatorio y un solo usuario CEO. No habra registro publico
ni multiusuario operativo.

El usuario CEO se crea por seed/setup. Las credenciales reales no deben quedar
documentadas en texto plano; se configuraran con secretos o hash:

- `CEO_EMAIL`
- `CEO_PASSWORD_HASH`

La sesion web usara JWT:

- access token de vida corta;
- refresh token o sesion persistente si se necesita renovar;
- claim `sub` con `user_id`;
- claim `role` con valor `CEO`;
- claim `session_id` si se conserva tabla de sesiones;
- firma con `JWT_SECRET` o par de llaves segun entorno.

Aunque solo exista el rol `CEO` en el MVP, el backend debe tener middleware de
autorizacion por rol para no acoplar seguridad a una suposicion de UI. CFO, COO
y lideres de area quedan fuera del MVP, pero el modelo no debe impedir agregarlos
despues.

### Data Ingestion and Chat Reporting

Para el MVP se trabajara con datos ficticios creados por seed de PostgreSQL. La
ingestion no representa una integracion productiva todavia; su objetivo es
alimentar respuestas ejecutivas realistas dentro del chatbot.

El flujo sera:

1. Prisma migration crea tablas fuente, views ejecutivas y tablas de auditoria.
2. Prisma seed inserta clientes, suscripciones, facturas, pipeline, proyectos,
   horas, tickets y gastos ficticios.
3. El chatbot consulta views gobernadas cuando el CEO pide un analisis.
4. El backend genera un artefacto de respuesta con narrativa, datos y
   visualizacion.
5. El artefacto queda asociado a la conversacion para preguntas de seguimiento.

Un artefacto de reporte conversacional debe incluir:

- `artifact_id`
- `conversation_id`
- `artifact_type`
- `question`
- `period`
- `source_views`
- `validated_sql`
- `summary`
- `data`
- `chart_spec`
- `freshness`
- `warnings`
- `trace_id`

Los dashboards persistentes, snapshots programados e historico formal de
reportes quedan como extension futura.

### LLM Orchestrator

El LLM Orchestrator es un modulo interno del backend. No es Codex ni Claude.
Codex y Claude son clientes posibles via MCP.

Responsabilidades:

- Recibir pregunta en lenguaje natural.
- Recibir `intent_mode` como complemento del prompt, no como filtro.
- Extraer requisitos explicitos del prompt.
- Construir `output_plan` combinando modo y requisitos explicitos.
- **Descomponer el prompt en un `execution_plan` tipado** (paso de Planner con el modelo
  ligero): sub-tareas `metric_query`, `knowledge_lookup` y `direct_answer`. Asi un prompt
  multi-intencion (metrica + conocimiento) se atiende en una sola respuesta. Ver
  [rag-knowledge-layer.md](rag-knowledge-layer.md).
- **Despachar las sub-tareas en paralelo:** `metric_query` al camino semantico/fallback;
  `knowledge_lookup` a la Capa de Conocimiento (RAG, read-only).
- **Sintetizar una sola respuesta** (modelo planificador) combinando artefactos de metrica
  y narrativa documental con citas; tratar el contenido recuperado como dato, no como
  instruccion.
- Cargar rol, permisos, schema y views permitidas.
- Cargar contexto conversacional minimo.
- Construir contexto para el LLM segun camino: `MetricCatalogContext` para
  `MetricQuery` y `BusinessSchemaContext` para fallback SQL. Ningun camino recibe DDL
  crudo completo.
- Pedir aclaraciones si la pregunta es ambigua.
- Camino por defecto: producir un `MetricQuery` validado contra el catalogo de
  metricas y dejar que la Capa Semantica compile el SQL de forma determinista.
- Camino de fallback: generar SQL candidato solo para preguntas exploratorias fuera
  de cobertura del catalogo (marcado como `path = fallback_sql`).
- Enviar el SQL (semantico o fallback) al SQL Safety Layer.
- Emitir log estructurado `warn` `analytics.fallback_sql_triggered` cada vez que se use
  fallback SQL, sin exponer datos sensibles.
- Ejecutar SQL validado.
- Generar respuesta ejecutiva.
- Proponer visualizacion.
- Persistir auditoria y artefactos conversacionales.
- Editar `chart_spec` de graficas existentes sin ejecutar SQL nuevo cuando solo
  cambie la configuracion visual.

### Capa Semantica (Metric Layer)

Capa headless ubicada entre el LLM Orchestrator y el SQL Safety Layer. Existe para
que las cifras ejecutivas sean exactas y para que el sistema **falle en voz alta**
(error o aclaracion) en vez de devolver un numero plausible pero incorrecto. El
diseno completo esta en `docs/architecture/semantic-layer-and-model-strategy.md` y la
decision en ADR-0006.

Piezas:

- **Catalogo de metricas** (YAML/JSON gobernado): define cada metrica oficial del MVP
  con `name`, `label`, `description`, `synonyms`, `grain`, `source_view` (mapea a las
  views `ceo_*`), `dimensions`, `filters_allowed`, `format` y `default_chart`. Funciona
  como documentacion, allowlist y contrato a la vez.
- **`MetricQuery`**: objeto estructurado que el LLM produce (`metric`, `dimensions`,
  `filters`, `time_range`, `compare_to`, `limit`). Se valida contra el catalogo antes
  de compilar nada; si no valida, se pide aclaracion o se cae al fallback.
- **Generador SQL determinista**: compila `MetricQuery` -> SQL sobre las views `ceo_*`.
  El LLM nunca escribe SQL en el camino semantico. El SQL generado igual pasa por el
  SQL Safety Layer.

El Text-to-SQL libre queda como **fallback** auditado para preguntas exploratorias
fuera del catalogo, sujeto a las mismas barreras. En ese camino el LLM recibe un
`BusinessSchemaContext` reducido (views `ceo_*`, columnas, relaciones, filtros y
funciones allowlisted), no la base cruda. Cada uso genera un log de desarrollo para
decidir si la pregunta debe promoverse al catalogo de metricas.

### Capa de Conocimiento (RAG)

Capa **hermana** de la Capa Semantica para preguntas **documentales no-metricas** (vision,
mision, a que se dedica la empresa, politicas, procesos). No produce SQL ni toca las views
`ceo_*`: recupera fragmentos (`chunks`) de documentos corporativos y los entrega al paso de
sintesis con citas. Diseno completo en [rag-knowledge-layer.md](rag-knowledge-layer.md) y
decision en ADR-0008.

Piezas:

- **Vector store (`pgvector`)**: extension sobre el PostgreSQL existente; tablas
  `documents` (registro/knowledge catalog) y `document_chunks` (embeddings + metadata). No
  se usa un servicio vectorial aparte, asi la recuperacion comparte el rol read-only y la
  auditoria.
- **Knowledge Retrieval** (modulo interno de `mirador-core`): embebe la consulta, hace
  busqueda vectorial top-k (read-only, filtrada por `access_scope`), rerank opcional y
  devuelve chunks con metadata de cita.
- **`KnowledgeBaseContext`**: indice compacto y cacheable de documentos disponibles que el
  Planner usa para decidir cuando enrutar a `knowledge_lookup`.

### Ingesta Documental (RAG)

La carga de documentos es **automatizada** y corre en un servicio asincrono independiente
(`mirador-ingestion`, Railway), separado del hot path. Es distinta del seed de datos
metricos: alimenta la Capa de Conocimiento, no las views `ceo_*`.

Plataformas: el archivo bruto vive en **Cloudflare R2**, pero la cola/trigger es **interna
de Railway** (tabla de jobs en PostgreSQL o Redis), no Cloudflare Queues. R2 expone API
**S3-compatible**, asi que `mirador-ingestion` lo lee por HTTPS con el SDK de S3, **sin
bindings de Cloudflare**. Los embeddings van a `pgvector` en el mismo Postgres de Railway.

Flujo: subida a R2 (via el servicio o URL prefirmada) -> el servicio encola un job en
Railway -> parseo por formato (PDF/Word/PPT/...) -> normalizacion -> chunking con overlap
-> embeddings -> upsert idempotente (`content_hash`) en `pgvector`. Re-indexa al reemplazar
y borra al eliminar. Ver [../diagrams/mermaid/rag-ingestion-flow.mmd](../diagrams/mermaid/rag-ingestion-flow.mmd).

### SQL Safety Layer

Valida el SQL antes de ejecutarlo (tanto el del camino semantico como el del fallback):

- Solo permite `SELECT`.
- Bloquea DDL, DML y multiples statements.
- Usa AST parser, no solo regex.
- Aplica allowlist de views, tablas, columnas y funciones.
- Aplica permisos segun rol.
- Fuerza `LIMIT`, max rows y timeout.
- Rechaza tablas internas no autorizadas.

### Database Layer

La base principal sera PostgreSQL serverless. Prisma ORM sera la capa de acceso
a datos para modelos tipados, migraciones y conexion limpia a PostgreSQL.

Se usaran dos credenciales:

- `DATABASE_URL_MIGRATION`: solo para migraciones/setup en entorno controlado.
- `DATABASE_URL_READONLY`: usada por Fastify/Prisma en runtime.

El usuario runtime debe ser read-only con permisos `SELECT` solo sobre views
analiticas o tablas permitidas. Railway Postgres sera la opcion favorita para el
MVP por compatibilidad directa con Prisma Client y el backend Fastify.

Views sugeridas:

- `ceo_revenue_summary`
- `ceo_customer_health`
- `ceo_sales_pipeline`
- `ceo_project_margin`
- `ceo_delivery_risk`
- `ceo_support_health`
- `ceo_financial_runway`

### Servicio MCP Independiente

El MCP ya no vive dentro de Fastify: es un **servicio desplegable propio** (Railway,
detras del MCP API Gateway). Funciona como **adapter delgado**: maneja el protocolo MCP,
valida `MCP_API_KEY` y el scope por tool, y mapea cada tool a la **API interna core** del
backend Fastify (`/internal/core/*`). El servicio MCP **no** accede a la base de datos ni
porta credenciales de DB o del proveedor LLM.

El endpoint MCP remoto (ahora en el host del servicio MCP) debe estar protegido:

```text
POST https://<host-mcp>/mcp
Authorization: Bearer <token>
```

Tools candidatas (cada una mapea a una llamada a la API interna core):

- `describe_business_schema`
- `ask_company_data`
- `run_readonly_query`
- `suggest_executive_questions`
- `generate_chart_spec`
- `search_company_knowledge` (RAG: recupera info documental, read-only, con citas)

Asi se reutiliza una sola Capa Semantica + SQL Safety Layer + acceso read-only + auditoria
(la logica de datos vive en la core API, no en el servicio MCP). La web SSR sigue
consumiendo `/api/auth/*`, `/api/chat/*` y `/api/schema/*` a traves del Web API Gateway;
el servicio MCP queda expuesto solo para clientes externos compatibles a traves del MCP
API Gateway.

## Flujo de Datos

### Flujo Login y JWT

1. El CEO abre `/login`.
2. Next.js envia credenciales a `POST /api/auth/login`.
3. Fastify valida hash de password del usuario seeded.
4. Fastify emite JWT con rol `CEO`.
5. Next.js guarda la sesion en cookie httpOnly o mecanismo equivalente.
6. El CEO entra a `/chat`.

### Flujo Chat-First

1. El CEO escribe una pregunta o selecciona una sugerencia.
2. Next.js envia mensaje a `POST /api/chat/messages` con `intent_mode`. La request pasa
   por el **Web API Gateway** (TLS, rate limiting, WAF, routing) antes de llegar a Fastify.
3. Fastify valida JWT, rol y sesion.
4. El LLM Orchestrator extrae requisitos explicitos del prompt y combina
   `mode_requirements` y `explicit_prompt_requirements` en un `output_plan`.
5. El Planner (modelo ligero) descompone el prompt en un `execution_plan` tipado
   (`metric_query` / `knowledge_lookup` / `direct_answer`) usando `MetricCatalogContext` y
   `KnowledgeBaseContext` (ambos cacheables).
6. El LLM Orchestrator carga contexto permitido y estado conversacional. Si falta
   informacion, el sistema pide aclaracion.
7. Las sub-tareas se despachan en **paralelo**:
   - `metric_query`: el LLM usa `MetricCatalogContext` y produce un `MetricQuery` validado
     contra el catalogo; la Capa Semantica lo compila a SQL de forma determinista. Si esta
     fuera de cobertura, usa `BusinessSchemaContext` y cae al fallback Text-to-SQL auditado.
   - `knowledge_lookup`: la Capa de Conocimiento embebe la consulta y recupera chunks top-k
     desde `pgvector` (read-only, filtrado por `access_scope`).
8. SQL Safety Layer valida por AST, allowlists, rol y limites; Database Layer ejecuta el
   camino de metrica con usuario read-only.
9. El paso de **sintesis** (modelo planificador) compone una sola respuesta: narrativa,
   tabla/grafica/KPI y narrativa documental con **citas**, segun `output_plan`. El
   contenido recuperado se trata como dato, no como instruccion.
10. Fastify registra auditoria (`execution_plan`, `retrieved_doc_ids`, `path`), emite log
    de fallback si aplica y devuelve `trace_id`.
11. Next.js renderiza los artefactos y las citas dentro del chat.

### Flujo de Edicion de Grafica

1. El CEO usa `Editar grafica` sobre un artefacto con `artifact_type = chart`.
2. Next.js abre un panel contextual con mini chat ligado a `artifact_id`.
3. Next.js envia instruccion a
   `POST /api/chat/artifacts/:artifact_id/chart-edits`.
4. Fastify valida JWT, rol, sesion y acceso al artefacto.
5. El backend aplica la instruccion sobre `current_chart_spec`.
6. Si la instruccion solo cambia visualizacion, responde con
   `updated_chart_spec`.
7. Si la instruccion pide cambiar datos, periodo, metrica o fuente, responde
   `requires_new_query: true` y deriva la solicitud al chat principal.
8. Cualquier nueva consulta de datos debe pasar por SQL Safety Layer.

### Flujo MCP

1. El CEO o usuario tecnico pregunta desde un cliente MCP externo.
2. La request pasa por el **MCP API Gateway** (TLS, rate limiting, cuotas por API key, WAF).
3. El **servicio MCP** valida `MCP_API_KEY` y el scope por tool, y mapea la tool a una
   llamada a la **API interna core** del backend (`POST /internal/core/ask`) con auth
   service-to-service.
4. El backend (LLM Orchestrator) descompone el prompt en un `execution_plan` y carga el
   catalogo de metricas permitido y el `KnowledgeBaseContext`.
5. Para `metric_query`, el LLM produce un `MetricQuery` si la pregunta esta cubierta por el
   catalogo; si no, usa `BusinessSchemaContext` para generar SQL candidato de fallback. Para
   `knowledge_lookup` (p. ej. tool `search_company_knowledge`), la Capa de Conocimiento
   recupera chunks desde `pgvector` (read-only).
6. SQL Safety Layer valida el SQL del camino de metrica.
7. Database Layer ejecuta con usuario read-only (metricas y busqueda vectorial).
8. El backend sintetiza la respuesta con citas, registra auditoria (`execution_plan`,
   `retrieved_doc_ids`), emite log de fallback si aplica y devuelve narrativa, datos,
   citas y warnings al servicio MCP, que responde al cliente con el formato MCP.

## Manejo del Contexto LLM

El contexto enviado al LLM debe incluir solo lo necesario y depende del camino:

- **Contexto comun:** rol del usuario, permisos, limites de seguridad, historial
  conversacional minimo y ultimo artefacto relevante, si aplica.
- **`MetricCatalogContext` (camino semantico):** catalogo de metricas permitido,
  definiciones, sinonimos, dimensiones, filtros, grano, `source_view`, formatos y reglas
  del contrato `MetricQuery`. El LLM selecciona metrica/intencion y devuelve
  `MetricQuery`; no escribe SQL.
- **`BusinessSchemaContext` (fallback SQL):** solo views analiticas `ceo_*`, columnas
  permitidas con descripcion de negocio, grano de cada view, relaciones/join paths
  permitidos, filtros, funciones allowlisted y reglas de seguridad. El LLM genera SQL
  candidato solo en este camino.
- **`KnowledgeBaseContext` (camino de conocimiento):** indice compacto y cacheable de los
  documentos disponibles (titulos, descripciones cortas, tags, `access_scope`). El
  contenido de los chunks solo se inyecta tras la recuperacion en el paso de sintesis, y se
  trata como dato, no como instruccion.

No debe incluir secretos, credenciales, DDL crudo completo, tablas bloqueadas,
relaciones no permitidas ni datos sensibles fuera del alcance.

### Estrategia de entrega de schema al LLM

No se vuelca el DDL completo. Se usan contextos gobernados (detalle en
`docs/architecture/semantic-layer-and-model-strategy.md`):

1. **`MetricCatalogContext` compacto** en el system prompt, estable entre interacciones
   y por tanto cacheable con prompt caching (amortiza su costo en tokens).
2. **`BusinessSchemaContext` para fallback**, solo cuando el catalogo de metricas no
   cubre la pregunta. Incluye metadata semantica allowlisted, no schema crudo.
3. **Detalle on-demand autenticado** via `GET /api/schema/catalog` o la tool MCP
   `describe_business_schema`, que devuelven el catalogo permitido por rol.

### Estrategia de modelos

La capa de modelos es agnostica de proveedor y se elige por configuracion
(`LLM_PROVIDER`, `ORCHESTRATOR_MODEL`, `LIGHT_MODEL`). Se usa **routing de dos niveles**:

- **Nivel ligero** (descomposicion del `execution_plan`, clasificacion de intencion,
  aclaraciones, `chart_spec`, sugerencias, ediciones visuales): modelo barato/rapido.
- **Nivel planificador** (NL -> `MetricQuery`, narrativa, analisis, fallback y **sintesis**
  final con citas): modelo fuerte en razonamiento estructurado.

Para RAG se agrega un tercer eje de configuracion, tambien agnostico de proveedor:
`EMBEDDING_PROVIDER` y `EMBEDDING_MODEL` (mismo modelo en ingesta y recuperacion). **Default
definido: `text-embedding-3-small` (OpenAI)** por coherencia de proveedor, con
`text-embedding-3-large` como upgrade; el costo RAG casi no depende del embedding (ver hoja
`RAG` de la calculadora). Cambiar de modelo de embeddings obliga a reindexar. Comparacion y
costos en [../cost/architecture-cost.md](../cost/architecture-cost.md).

**Modelos definidos (default): GPT-5.2 como planificador y GPT-5 mini como nivel
ligero** (ambos OpenAI). Se eligieron cruzando exactitud (la familia GPT-5.x lidera el
benchmark de capa semantica de dbt) y costo (la calculadora `docs/cost/`): GPT-5.2 corre
~22% mas barato por interaccion que Claude Sonnet 4.6 y tiene la tarifa de input cacheado
mas baja, clave para nuestra estrategia de prompt caching. La capa de modelos es
intercambiable por configuracion (`ORCHESTRATOR_MODEL`, `LIGHT_MODEL`); Gemini 2.5
Pro/Flash es la alternativa mas barata y Claude Sonnet/Haiku la de mayor exactitud
publicada no-Codex. Analisis y fuentes en `docs/cost/README.md` y
`docs/architecture/semantic-layer-and-model-strategy.md`.

## Seguridad

- Borde de **dos API Gateways Cloudflare** (web y MCP) con rate limiting, throttling,
  cuotas, WAF/IP rules y limite de tamano antes de llegar a los servicios y al pipeline LLM.
- Los gateways hacen auth coarse + control de trafico; la authz fina queda en los servicios.
- Autenticacion obligatoria para web y MCP.
- JWT para sesion web.
- Bearer token/API key para MCP remoto.
- Login web con usuario CEO seeded; sin registro publico.
- Middleware de autorizacion por rol.
- Rol PostgreSQL read-only.
- Prisma Client usando `DATABASE_URL_READONLY` en runtime.
- Migraciones con `DATABASE_URL_MIGRATION` fuera del runtime de consulta.
- SQL Safety Layer antes de ejecutar.
- Allowlist de schema.
- `LIMIT`, timeout y max rows obligatorios.
- Auditoria de prompts, `path`, `MetricQuery`, `execution_plan`, `retrieved_doc_ids`,
  razon de fallback, SQL candidato, SQL validado, cliente, usuario y `trace_id`.
- **Capa de Conocimiento (RAG) read-only:** la recuperacion vectorial usa el mismo rol
  read-only de PostgreSQL y filtra por `access_scope`/rol.
- **Mitigacion de prompt-injection:** el contenido recuperado de documentos se trata como
  dato, no como instruccion; se sanitiza antes de inyectarlo y el system prompt de sintesis
  esta endurecido. Sin evidencia suficiente, la respuesta dice que no se encontro.
- Log estructurado `warn` `analytics.fallback_sql_triggered` cuando el sistema usa
  fallback SQL; debe usar hashes o contenido sanitizado si el SQL contiene valores
  sensibles.
- No exponer schema sin autenticacion.
- Separar tokens MCP de credenciales DB y LLM.
- Separar auth web de MCP; el frontend no debe usar `MCP_API_KEY`.
- El **servicio MCP no porta credenciales de DB ni del proveedor LLM**: solo llama a la
  API interna core. Las credenciales de DB/LLM viven solo en el backend core.
- La **API interna core** (`/internal/core/*`) se protege con defensa en profundidad: la
  **red privada de Railway** (`*.railway.internal`) como frontera primaria (sin dominio
  publico, no enrutada por el Web API Gateway) y el **`CORE_SERVICE_TOKEN`** como segunda
  capa, validado en cada request. El backend rechaza `/internal/*` por interfaz publica.
- No usar Prisma como unica barrera de seguridad; las raw queries generadas por
  IA solo pueden ejecutarse despues de validacion AST.
- Las ediciones de grafica no ejecutan SQL si solo modifican `chart_spec`.
- Cambios de datos, periodo, metrica o fuente deben crear una nueva solicitud de
  chat y volver a pasar por SQL Safety Layer.

## Respuesta Comun

La web y MCP deben compartir una estructura:

### Request de Chat

`POST /api/chat/messages` recibe:

```json
{
  "conversation_id": "conversation_uuid",
  "content": "Analiza la caida de MRR e incluye una grafica",
  "intent_mode": "analizar",
  "context_artifact_id": "artifact_uuid"
}
```

El backend deriva:

```json
{
  "mode_requirements": ["analysis", "evidence"],
  "explicit_prompt_requirements": ["chart"],
  "output_plan": ["text", "chart"]
}
```

`intent_mode` puede ser `responder`, `analizar`, `reporte_visual` o `plan`.
El modo define la prioridad base, pero los requisitos explicitos del prompt se
combinan con el modo y no se descartan.

### Response de Chat

```json
{
  "answer": "Resumen ejecutivo en lenguaje natural.",
  "sql": "SELECT ...",
  "data": [],
  "artifacts": [],
  "citations": [
    {
      "document_id": "doc_uuid",
      "title": "Manifiesto de la empresa 2026",
      "locator": "pag. 2",
      "snippet": "Nuestra mision es..."
    }
  ],
  "chart": {
    "type": "line",
    "x": "month",
    "y": "mrr"
  },
  "warnings": [],
  "suggested_questions": [],
  "metadata": {
    "client_type": "web",
    "rows": 12,
    "execution_plan": ["metric_query", "knowledge_lookup"]
  },
  "trace_id": "uuid"
}
```

`citations` lista las fuentes documentales que respaldan la narrativa cuando el
`execution_plan` incluye `knowledge_lookup`; queda vacio en respuestas solo de metrica.
Los valores permitidos de `artifact_type` para MVP incluyen ademas `knowledge` para la
narrativa documental con citas.

Para artefactos generados en chat se extiende con:

```json
{
  "artifact_id": "artifact_uuid",
  "conversation_id": "conversation_uuid",
  "artifact_type": "chart",
  "question": "Mostrar MRR y crecimiento de los ultimos 6 meses",
  "period": "last_6_months",
  "source_views": ["ceo_revenue_summary"],
  "freshness": {
    "generated_at": "2026-06-01T08:00:00Z",
    "status": "fresh"
  },
  "summary": "MRR crecio 8% en los ultimos 6 meses.",
  "chart_spec": {
    "type": "line",
    "x": "month",
    "y": "mrr"
  }
}
```

Los valores permitidos de `artifact_type` para MVP son `text`, `table`, `kpi`,
`chart`, `report` y `action_plan`.

### Request de Edicion de Grafica

`POST /api/chat/artifacts/:artifact_id/chart-edits` recibe:

```json
{
  "instruction": "Cambiar a barras y resaltar marzo",
  "current_chart_spec": {
    "type": "line",
    "x": "month",
    "y": "mrr"
  }
}
```

Responde:

```json
{
  "updated_chart_spec": {
    "type": "bar",
    "x": "month",
    "y": "mrr",
    "annotations": [
      {
        "x": "2026-03",
        "label": "Caida relevante"
      }
    ]
  },
  "change_summary": "Se cambio la grafica a barras y se anoto marzo.",
  "warnings": [],
  "requires_new_query": false
}
```

## Actualizacion de Reportes

El MVP no promete tiempo real ni reportes programados. Los datos se alimentan por
seed y se consultan bajo demanda desde el chatbot. Si una consulta es costosa o
frecuente, puede introducirse cache o snapshot conversacional, pero no como
dashboard principal.

## Entregables de Fase 2

Cuando se apruebe la arquitectura y se desarrolle la solucion, los entregables
seran:

- URL de aplicacion web.
- URL del servidor MCP.
- API key o token valido para MCP.
- Instrucciones de configuracion para cliente MCP.
- Credenciales de prueba o usuario demo para staging.
