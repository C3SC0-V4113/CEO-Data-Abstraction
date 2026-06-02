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
Frontend (Cloudflare Workers / Next.js), Backend (Railway / Fastify), Datos (Railway
PostgreSQL) y Servicios externos (LLM Provider y clientes MCP), igual que en los Mermaid.
Nombrar el modelo LLM: **GPT-5.2** (planificador) y **GPT-5 mini** (ligero).

## architecture (drawio/architecture.drawio, xml/architecture.drawio.xml)

Fuente: `mermaid/architecture.mmd`. Nodos y aristas:

- CEO -> Browser -> Cloudflare Workers (Next.js SSR, login + chatbot).
- Cloudflare -> Fastify (Railway) por HTTP: `/api/auth/*`, `/api/chat/*`,
  `/api/schema/catalog`. Arista punteada Cloudflare -.- MCP con nota
  "does not call MCP directly".
- External MCP clients (Claude Desktop / Cursor / Codex) -> MCP `POST /mcp` con
  "Bearer token / MCP_API_KEY" -> Fastify.
- Fastify -> LLM Orchestrator -> (a) LLM Provider con "compact metric catalog
  (cacheable)"; (b) Semantic Metric Layer con "MetricQuery (default) or fallback SQL".
- Semantic Metric Layer -> SQL Safety Layer (AST, allowlists, SELECT only, LIMIT,
  timeout) -> Prisma ORM -> PostgreSQL (read-only, `ceo_*` views).
- Prisma seed -> PostgreSQL. Fastify y MCP -> query_audit_log (path: semantic |
  fallback_sql).
- Eliminar el nodo "Report snapshots / report_definitions / report_snapshots /
  dashboard_widgets" y las rutas `/api/dashboard/*` y `/api/reports/*`.

## use-cases-and-flows (drawio/use-cases-and-flows.drawio, xml/use-cases-and-flows.drawio.xml)

Fuente: `mermaid/use-cases.mmd`. CEO -> Login -> Chat; desde Chat: Clarify,
MetricQuery (semantic) y Fallback Text-to-SQL; ambos -> SQL Safety Layer -> Chat
artifact -> Edit chart (visual). Artifact -> Audit. External MCP client -> MCP query
-> MetricQuery -> Audit. Eliminar "View executive dashboard" y "Filter report".

## frontend-flow (drawio/frontend-flow.drawio, xml/frontend-flow.drawio.xml)

Fuente: `mermaid/frontend-flow.mmd`. Secuencia login -> chat -> POST
/api/chat/messages -> Orchestrator -> MetricQuery -> Semantic Layer -> SQL Safety
Layer -> PostgreSQL read-only -> artifact renderizado en el hilo -> mini-chat de
edicion de grafica (`/api/chat/artifacts/:id/chart-edits`, sin SQL nuevo si solo es
visual). Eliminar "Open /dashboard", "GET /api/dashboard/overview" y "report
snapshots".

## mcp-flow (drawio/mcp-flow.drawio, xml/mcp-flow.drawio.xml)

Fuente: `mermaid/mcp-flow.mmd`. Igual al actual pero insertando el paso
Orchestrator -> Semantic Metric Layer (MetricQuery) antes del SQL Safety Layer, y el
registro de auditoria con `path` y `metric_query`.

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
