# Diagrams

Esta carpeta contiene los diagramas fuente y editables para presentar la
arquitectura, modelo de datos, flujos y wireframes del MVP.

## Estado

Los diagramas actuales fueron creados antes del cambio a experiencia
chat-first. La arquitectura textual vigente esta en
`docs/architecture/proposal.md` y ADR-0005. Estos diagramas deben regenerarse
despues de estabilizar la arquitectura textual.

## Estructura

```text
docs/diagrams/
  drawio/   Archivos .drawio editables en diagrams.net.
  mermaid/  Fuentes Mermaid para revision textual.
  xml/      XML draw.io/mxGraph para abrir con MCP o diagrams.net.
  assets/   Politica de logos e imagenes.
```

## Diagramas Pendientes de Realinear

- [architecture](drawio/architecture.drawio): pendiente de actualizar a login,
  chatbot, Fastify, Prisma, PostgreSQL, MCP externo, LLM y SQL Safety Layer.
- [database-model](drawio/database-model.drawio): pendiente de actualizar con
  conversaciones, mensajes, artefactos de chat, auth, auditoria y views
  `ceo_*`.
- [use-cases-and-flows](drawio/use-cases-and-flows.drawio): pendiente de
  actualizar a flujo login -> chat -> artefacto.
- [frontend-flow](drawio/frontend-flow.drawio): pendiente de actualizar a flujo
  SSR de login y chatbot.
- [mcp-flow](drawio/mcp-flow.drawio): flujo externo MCP con bearer token,
  orquestador, SQL Safety Layer y PostgreSQL.
- [ui-wireframes](drawio/ui-wireframes.drawio): pendiente de actualizar a login
  y chatbot con artefactos embebidos.

## Flujo Recomendado

1. Mantener los Mermaid como version ligera para revision.
2. Mantener XML/.drawio como version editable en draw.io.
3. Usar el MCP de draw.io en Codex para abrir Mermaid/XML cuando este instalado.
4. Si se hacen cambios visuales en draw.io, exportar de vuelta a `.drawio`.
