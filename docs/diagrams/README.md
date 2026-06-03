# Diagrams

Esta carpeta contiene los diagramas fuente y editables para presentar la
arquitectura, modelo de datos, flujos y wireframes del MVP.

## Estado

Los **Mermaid** (`mermaid/*.mmd`) ya estan actualizados a la experiencia chat-first
con capa semantica y son la **fuente de verdad textual**. La arquitectura vigente esta
en `docs/architecture/proposal.md`, ADR-0005 y ADR-0006.

Los Mermaid estan **agrupados por infraestructura/proyecto**: Cliente, Frontend
(Cloudflare Workers / Next.js), **Borde - API Gateways (Cloudflare)** (Web Gateway y MCP
Gateway), **Servicio MCP (Railway, proyecto independiente)**, Backend core (Railway /
Fastify, con la Core Internal API), Datos (Railway PostgreSQL) y Servicios externos
(LLM Provider y clientes MCP). Los flowcharts usan `subgraph` y los diagramas de
secuencia usan `box` (requiere Mermaid >= 9.4; soportado por mermaid.live y GitHub). El
modelo LLM definido aparece nombrado: **GPT-5.2** (planificador) y **GPT-5 mini** (ligero).

Los **drawio/xml** todavia reflejan el paradigma report-first y estan **pendientes de
regenerar** a partir de los Mermaid. Las instrucciones por archivo estan en
`docs/diagrams/REGENERATION-SPEC.md`.

## Estructura

```text
docs/diagrams/
  drawio/   Archivos .drawio editables en diagrams.net.
  mermaid/  Fuentes Mermaid para revision textual.
  xml/      XML draw.io/mxGraph para abrir con MCP o diagrams.net.
  assets/   Politica de logos e imagenes.
```

## Diagramas Pendientes de Realinear (drawio/xml)

Ver el detalle por archivo (nodos, aristas, que eliminar) en
[REGENERATION-SPEC.md](REGENERATION-SPEC.md). Resumen:

- [architecture](drawio/architecture.drawio): login, chatbot, Fastify, LLM
  Orchestrator, **Capa Semantica/Metric Layer**, SQL Safety Layer, Prisma,
  PostgreSQL `ceo_*`, MCP externo. Separa `MetricCatalogContext` (camino semantico) y
  `BusinessSchemaContext` (fallback SQL). Sin dashboards/reportes/snapshots.
- [semantic-layer-internal-flow](mermaid/semantic-layer-internal-flow.mmd): flujo interno
  de la Metric Layer, rol del LLM, `MetricQuery`, fallback SQL gobernado, log
  `analytics.fallback_sql_triggered` y promocion de preguntas recurrentes al catalogo.
- [database-model](drawio/database-model.drawio): conversations, chat_messages,
  chat_artifacts, auth, auditoria con `path`/`metric_query`. Sin
  report_definitions/report_snapshots/dashboard_widgets.
- [use-cases-and-flows](drawio/use-cases-and-flows.drawio): login -> chat ->
  MetricQuery/fallback -> SQL Safety Layer -> artefacto.
- [frontend-flow](drawio/frontend-flow.drawio): SSR login + chatbot, MetricQuery,
  capa semantica, mini-chat de grafica.
- [mcp-flow](drawio/mcp-flow.drawio): MCP externo con bearer token, orquestador,
  capa semantica, SQL Safety Layer y PostgreSQL.
- [ui-wireframes](drawio/ui-wireframes.drawio): solo login y chatbot con artefactos
  embebidos y mini-chat de grafica.

## Flujo Recomendado

1. Mantener los Mermaid como version ligera para revision.
2. Mantener XML/.drawio como version editable en draw.io.
3. Usar el MCP de draw.io en Codex para abrir Mermaid/XML cuando este instalado.
4. Si se hacen cambios visuales en draw.io, exportar de vuelta a `.drawio`.
