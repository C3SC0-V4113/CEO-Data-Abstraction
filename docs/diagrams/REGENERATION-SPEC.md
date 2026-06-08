# Especificacion de Regeneracion de Diagramas (drawio/xml)

Los `.mmd` de `docs/diagrams/mermaid/` ya estan actualizados a chat-first + capa
semantica y son la **fuente de verdad textual**. Los `.drawio` y `.xml` siguen
mostrando el paradigma report-first (dashboards, reportes, snapshots) y deben
regenerarse para coincidir con los Mermaid, `docs/architecture/proposal.md` y
ADR-0005/0006.

Regla general para todos: eliminar dashboards, rutas de reportes, snapshots y widgets
como superficies; el reporte es un **artefacto dentro del chat**. Insertar la
**Capa Semantica (Metric Layer)** entre el LLM Orchestrator y el SQL Safety Layer.
Separar visualmente el camino principal (`MetricCatalogContext` -> LLM -> `MetricQuery`
-> SQL determinista) del fallback (`BusinessSchemaContext` -> LLM -> SQL candidato).
Cada fallback debe mostrar el log `analytics.fallback_sql_triggered` conectado a
auditoria y discovery para ampliar el catalogo.
**Agrupar los nodos por infraestructura/proyecto** (contenedores en draw.io): Cliente,
Frontend (Cloudflare Workers / Next.js), **Borde - API Gateways (Cloudflare)** (Web Gateway
+ MCP Gateway), **Servicio MCP (Railway, proyecto independiente)**, Backend core (Railway /
Fastify, con la Core Internal API `/internal/core/*`), Datos (Railway PostgreSQL) y Servicios
externos (LLM Provider y clientes MCP), igual que en los Mermaid. Nombrar el modelo LLM:
**GPT-5.2** (planificador) y **GPT-5 mini** (ligero). Reflejar que el trafico entra por los
gateways y que el servicio MCP es un adapter que llama a la Core Internal API (no toca la DB).

## architecture (drawio/architecture.drawio, xml/architecture.drawio.xml)

Fuente: `mermaid/architecture.mmd`. Nodos y aristas:

- CEO -> Browser -> Cloudflare Workers (Next.js SSR, login + chatbot).
- Next.js -> **Web API Gateway (Cloudflare)** por HTTP (`/api/auth/*`, `/api/chat/*`,
  `/api/schema/catalog`) -> backend Fastify (LLM Orchestrator).
- External MCP clients (Claude Desktop / Cursor / Codex) -> **MCP API Gateway
  (Cloudflare)** con "Bearer MCP_API_KEY" -> **Servicio MCP (Railway, adapter)** ->
  **Core Internal API** del backend (`/internal/core/*`, `CORE_SERVICE_TOKEN`).
- Backend Fastify: Core Internal API -> LLM Orchestrator -> (a) `MetricCatalogContext`
  -> LLM Provider (Planner GPT-5.2 / Light GPT-5 mini) -> `MetricQuery` -> Semantic
  Metric Layer; (b) `BusinessSchemaContext` -> LLM Provider -> Fallback Text-to-SQL
  SQL candidate.
- Semantic Metric Layer -> SQL Safety Layer (AST, allowlists, SELECT only, LIMIT,
  timeout) -> Prisma ORM -> PostgreSQL (read-only, `ceo_*` views).
- Fallback Text-to-SQL -> SQL Safety Layer y -> `warn log:
  analytics.fallback_sql_triggered` -> query_audit_log.
- Prisma seed -> PostgreSQL. Orchestrator -> query_audit_log (path: semantic |
  fallback_sql). El Servicio MCP **no** toca la DB.
- Contenedores por infra: Cliente, Frontend (Cloudflare), Borde - API Gateways
  (Cloudflare), Servicio MCP (Railway), Backend core (Railway/Fastify), Datos (Railway
  PostgreSQL), Servicios externos.
- Eliminar el nodo "Report snapshots / report_definitions / report_snapshots /
  dashboard_widgets" y las rutas `/api/dashboard/*` y `/api/reports/*`.

## use-cases-and-flows (drawio/use-cases-and-flows.drawio, xml/use-cases-and-flows.drawio.xml)

Fuente: `mermaid/use-cases.mmd`. CEO -> Login -> Chat -> **Web API Gateway** -> {Clarify,
MetricQuery (semantic), Fallback Text-to-SQL}; ambos -> SQL Safety Layer -> Chat artifact
-> Edit chart (visual); Fallback Text-to-SQL -> log `analytics.fallback_sql_triggered`
-> Audit. External MCP client -> **MCP API Gateway** ->
**Servicio MCP** -> **Core Internal API** -> MetricQuery. Agrupar por infra (Borde,
Servicio MCP, Backend core). Eliminar "View executive dashboard" y "Filter report".

## semantic-layer-internal-flow (mermaid/semantic-layer-internal-flow.mmd)

Fuente textual nueva para explicar internamente la Metric Layer. Si se genera version
draw.io, debe mostrar:

- Pregunta CEO -> LLM Orchestrator -> LLM Planner (GPT-5.2).
- Camino principal: `MetricCatalogContext` -> `MetricQuery` -> Catalog Validator ->
  Deterministic SQL Compiler -> SQL Safety Layer -> PostgreSQL read-only.
- Camino fallback: fuera de cobertura -> `BusinessSchemaContext` -> LLM SQL Candidate ->
  SQL Safety Layer.
- Fallback log `analytics.fallback_sql_triggered` -> query_audit_log -> Discovery backlog
  para ampliar el catalogo.
- Recalcar que el LLM escribe SQL solo en fallback; en el camino semantico el LLM produce
  `MetricQuery`.

## frontend-flow (drawio/frontend-flow.drawio, xml/frontend-flow.drawio.xml)

Fuente: `mermaid/frontend-flow.mmd`. Secuencia login -> chat -> POST
/api/chat/messages -> **Web API Gateway** -> Fastify API -> Orchestrator -> LLM
(GPT-5.2) con `MetricCatalogContext` -> MetricQuery -> Semantic Layer -> SQL Safety
Layer -> PostgreSQL read-only. Incluir rama alternativa: Orchestrator ->
`BusinessSchemaContext` -> LLM SQL candidate -> Fallback Text-to-SQL -> SQL Safety Layer
-> log `analytics.fallback_sql_triggered`. Luego artifact renderizado en el hilo ->
mini-chat de edicion de grafica
(`/api/chat/artifacts/:id/chart-edits`, sin SQL nuevo si solo es visual). Agrupar los
participantes por infra con cajas (Cloudflare, Borde, Railway Fastify, Datos, Externo).
Eliminar "Open /dashboard", "GET /api/dashboard/overview" y "report snapshots".

## mcp-flow (drawio/mcp-flow.drawio, xml/mcp-flow.drawio.xml)

Fuente: `mermaid/mcp-flow.mmd`. Cliente MCP -> **MCP API Gateway** -> **Servicio MCP**
(valida MCP_API_KEY + scope) -> **Core Internal API** (`CORE_SERVICE_TOKEN`) -> Orchestrator
-> LLM (GPT-5.2) con `MetricCatalogContext` -> Semantic Metric Layer (`MetricQuery`) ->
SQL Safety Layer -> Prisma -> PostgreSQL read-only. Incluir rama fallback con
`BusinessSchemaContext`, SQL candidate y log `analytics.fallback_sql_triggered`.
Auditoria con `path`, `metric_query`, razon de fallback y SQL en el backend. El Servicio
MCP no toca la DB. Agrupar participantes por infra con cajas (Externo, Borde, Servicio
MCP, Backend core, Datos).

## database-model (drawio/database-model.drawio, xml/database-model.drawio.xml)

Fuente: `mermaid/database-model.mmd`. Mantener tablas fuente (customers,
subscriptions, invoices, sales_opportunities, projects, time_entries,
support_tickets, expenses, users, sessions). Reemplazar report_definitions,
report_snapshots, dashboard_widgets, report_jobs y metric_snapshots por
**conversations**, **chat_messages** y **chat_artifacts**. En query_audit_log agregar
`path` y `metric_query`. Las views `ceo_*` se derivan de las tablas fuente; el
catalogo de metricas es configuracion (YAML/JSON), no tabla.

## ui-wireframes (drawio/ui-wireframes.drawio, xml/ui-wireframes.drawio.xml)

Fuente: `mermaid/ui-wireframes.mmd`. Solo dos pantallas: Login y Chatbot. El Chatbot
incluye composer con modos (responder/analizar/reporte_visual/plan), preguntas
sugeridas, artefactos inline (text/kpi/table/chart con freshness/warnings/trace_id) y
mini-chat de grafica. Eliminar Dashboard, Revenue view, Projects view y Reports
history.

## Archivos `*_mermaid.drawio`

`architecture_mermaid.drawio`, `frontend_flow_mermaid.drawio` y
`mcp_flow_mermaid.drawio` fueron importados desde los Mermaid anteriores;
regenerarlos importando los `.mmd` actualizados.

## Adiciones RAG (ADR-0008)

Los `.mmd` ya incluyen la capa de Conocimiento (RAG) y la orquestacion multi-intencion;
los `.drawio`/`.xml` deben incorporarlas al regenerar. Resumen por diagrama:

- **architecture**: agregar contenedor **Ingesta RAG - Railway (`ceo-chat-ingestion`)** y
  **Object storage - Cloudflare R2**; dentro del backend core, los nodos **Planner**
  (descompone `execution_plan`), **Knowledge Retrieval**, **KnowledgeBaseContext** y
  **Sintesis**; en Datos, **documents / document_chunks (pgvector)**. Aristas:
  Orchestrator -> Planner -> {Semantic | Knowledge Retrieval}; Knowledge Retrieval ->
  Prisma -> pgvector; Semantic + Knowledge Retrieval -> Sintesis -> respuesta; R2 ->
  Ingesta -> pgvector. `query_audit_log` ahora lleva `execution_plan` y `retrieved_doc_ids`.
- **use-cases-and-flows**: agregar caso "consultar info documental (vision/mision)" y el
  caso multi-intencion (metrica + conocimiento en una respuesta) via Planner -> {Metric,
  Knowledge, Fallback, Clarify} -> Sintesis -> artifact.
- **semantic-layer-internal-flow**: anteponer la descomposicion del Planner (LIGHT_MODEL)
  y mostrar el camino `knowledge_lookup` (Knowledge Retrieval -> document_chunks) como
  hermano del semantico/fallback, convergiendo en Sintesis.
- **frontend-flow** y **mcp-flow**: agregar el paso de Planner y un bloque `par`
  (metric_query y knowledge_lookup en paralelo) que converge en Sintesis con citas. En MCP,
  nombrar la tool `search_company_knowledge`.
- **database-model**: agregar `documents` y `document_chunks` (con `embedding` vector de
  pgvector) y los campos `execution_plan` / `retrieved_doc_ids` en `query_audit_log`.

Diagramas nuevos sin equivalente report-first (no requieren limpieza, solo generar
`.drawio`/`.xml`): `rag-ingestion-flow.mmd` y `rag-multi-intent-orchestration.mmd`.

**Comunicacion R2 <-> Railway (importante para no dibujar mal la ingesta):** el archivo
bruto vive en Cloudflare R2 y se accede por **API S3** desde Railway (sin bindings); la
**cola/trigger es interna de Railway** (jobs en PostgreSQL o Redis), **no** Cloudflare
Queues. No dibujar un Worker-puente ni una Cloudflare Queue entre R2 y `ceo-chat-ingestion`.

**Modelo de embeddings:** los nodos de embedding (ingesta y embed-query) nombran el modelo
definido **`text-embedding-3-small`** (OpenAI; default por coherencia de proveedor, ver
hoja `RAG` de `docs/cost/llm-cost-calculator.xlsx`). `text-embedding-3-large` es el upgrade
de calidad. Reflejarlo igual en los `.drawio`.

## Como regenerar

1. Abrir el `.mmd` actualizado en draw.io (Arrange -> Insert -> Advanced -> Mermaid)
   o con el MCP de draw.io (ver `docs/diagrams/drawio-mcp.md`).
2. Ajustar layout y exportar a `.drawio` y `.xml` con el mismo nombre base.
3. Verificar que ningun diagrama muestre dashboards, rutas de reportes o snapshots
   como superficie.
