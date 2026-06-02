# Especificacion de Regeneracion de Diagramas (drawio/xml)

Los `.mmd` de `docs/diagrams/mermaid/` ya estan actualizados a chat-first + capa
semantica y son la **fuente de verdad textual**. Los `.drawio` y `.xml` siguen
mostrando el paradigma report-first (dashboards, reportes, snapshots) y deben
regenerarse para coincidir con los Mermaid, `docs/architecture/proposal.md` y
ADR-0005/0006.

Regla general para todos: eliminar dashboards, rutas de reportes, snapshots y widgets
como superficies; el reporte es un **artefacto dentro del chat**. Insertar la
**Capa Semantica (Metric Layer)** entre el LLM Orchestrator y el SQL Safety Layer.
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
- Backend Fastify: Core Internal API -> LLM Orchestrator -> (a) LLM Provider
  (Planner GPT-5.2 / Light GPT-5 mini) con "compact metric catalog (cacheable)";
  (b) Semantic Metric Layer con "MetricQuery (default) or fallback SQL".
- Semantic Metric Layer -> SQL Safety Layer (AST, allowlists, SELECT only, LIMIT,
  timeout) -> Prisma ORM -> PostgreSQL (read-only, `ceo_*` views).
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
-> Edit chart (visual); Artifact -> Audit. External MCP client -> **MCP API Gateway** ->
**Servicio MCP** -> **Core Internal API** -> MetricQuery. Agrupar por infra (Borde,
Servicio MCP, Backend core). Eliminar "View executive dashboard" y "Filter report".

## frontend-flow (drawio/frontend-flow.drawio, xml/frontend-flow.drawio.xml)

Fuente: `mermaid/frontend-flow.mmd`. Secuencia login -> chat -> POST
/api/chat/messages -> **Web API Gateway** -> Fastify API -> Orchestrator -> LLM
(GPT-5.2) -> MetricQuery -> Semantic Layer -> SQL Safety Layer -> PostgreSQL read-only
-> artifact renderizado en el hilo -> mini-chat de edicion de grafica
(`/api/chat/artifacts/:id/chart-edits`, sin SQL nuevo si solo es visual). Agrupar los
participantes por infra con cajas (Cloudflare, Borde, Railway Fastify, Datos, Externo).
Eliminar "Open /dashboard", "GET /api/dashboard/overview" y "report snapshots".

## mcp-flow (drawio/mcp-flow.drawio, xml/mcp-flow.drawio.xml)

Fuente: `mermaid/mcp-flow.mmd`. Cliente MCP -> **MCP API Gateway** -> **Servicio MCP**
(valida MCP_API_KEY + scope) -> **Core Internal API** (`CORE_SERVICE_TOKEN`) -> Orchestrator
-> LLM (GPT-5.2) -> Semantic Metric Layer (MetricQuery) -> SQL Safety Layer -> Prisma ->
PostgreSQL read-only; auditoria con `path` y `metric_query` en el backend. El Servicio MCP
no toca la DB. Agrupar participantes por infra con cajas (Externo, Borde, Servicio MCP,
Backend core, Datos).

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

## Como regenerar

1. Abrir el `.mmd` actualizado en draw.io (Arrange -> Insert -> Advanced -> Mermaid)
   o con el MCP de draw.io (ver `docs/diagrams/drawio-mcp.md`).
2. Ajustar layout y exportar a `.drawio` y `.xml` con el mismo nombre base.
3. Verificar que ningun diagrama muestre dashboards, rutas de reportes o snapshots
   como superficie.
