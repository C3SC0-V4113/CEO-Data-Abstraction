# Diagrams

Esta carpeta contiene los diagramas fuente y editables para presentar la
arquitectura, modelo de datos, flujos y wireframes del MVP.

## Estructura

```text
docs/diagrams/
  drawio/   Archivos .drawio editables en diagrams.net.
  mermaid/  Fuentes Mermaid para revision textual.
  xml/      XML draw.io/mxGraph para abrir con MCP o diagrams.net.
  assets/   Politica de logos e imagenes.
```

## Diagramas

- [architecture](drawio/architecture.drawio): despliegue Cloudflare/Railway,
  Fastify, Prisma, PostgreSQL, MCP externo, LLM, SQL Safety Layer y reportes.
- [database-model](drawio/database-model.drawio): tablas fuente, auth, reportes,
  snapshots, auditoria y views `ceo_*`.
- [use-cases-and-flows](drawio/use-cases-and-flows.drawio): casos de uso, flujo
  frontend y flujo MCP.
- [frontend-flow](drawio/frontend-flow.drawio): flujo SSR del dashboard y chat
  contextual.
- [mcp-flow](drawio/mcp-flow.drawio): flujo externo MCP con bearer token,
  orquestador, SQL Safety Layer y PostgreSQL.
- [ui-wireframes](drawio/ui-wireframes.drawio): login, dashboard, reportes,
  graficas y chat contextual.

## Flujo Recomendado

1. Mantener los Mermaid como version ligera para revision.
2. Mantener XML/.drawio como version editable en draw.io.
3. Usar el MCP de draw.io en Codex para abrir Mermaid/XML cuando este instalado.
4. Si se hacen cambios visuales en draw.io, exportar de vuelta a `.drawio`.
