# CLAUDE.md

Guia para Claude Code en este repositorio. Las instrucciones operativas viven en
[AGENTS.md](AGENTS.md): respetalo como fuente principal. Este archivo solo agrega
contexto rapido y no debe contradecirlo.

## Reglas heredadas de AGENTS.md (resumen)

- Documentacion en espanol claro y tecnico; tono pragmatico:
  discovery -> decision -> implementacion.
- Priorizar discovery, estrategia y ADRs antes de proponer codigo.
- No permitir SQL libre generado por un LLM como recomendacion por defecto: el SQL
  debe pasar por el SQL Safety Layer y ejecutarse con rol PostgreSQL read-only.
- Stack vigente: Next.js SSR + shadcn/ui, Fastify + TypeScript, Prisma ORM, MCP
  remoto, PostgreSQL serverless, despliegue Cloudflare o Railway.
- Las decisiones se registran en `docs/adr/` (formato `docs/adr/template.md`); no
  reescribir un ADR aceptado, crear uno que lo superseda.

## Estado del proyecto

Fase 1: repositorio **solo de documentacion**. No hay codigo de aplicacion, base de
datos ni MCP server todavia. El paradigma vigente es **chatbot-first** (ADR-0005):
login + chatbot como unicas interfaces; reportes generados como artefactos dentro
del chat, no como dashboards o vistas separadas. El chat atiende metricas (Capa
Semantica, ADR-0006) y preguntas documentales (Capa de Conocimiento / RAG, ADR-0008),
con un planner que enruta ambas y las combina en una sola respuesta.

## Mapa del repositorio

```text
AGENTS.md                Instrucciones para agentes (fuente principal).
docs/adr/                Decisiones de arquitectura + plantilla.
docs/architecture/       proposal.md (arquitectura maestra) y disenos de detalle.
docs/strategy/           Opciones de solucion y lineas de brainstorming.
docs/discovery/          Problema, usuario CEO, casos de uso, datos asumidos, dudas.
docs/glossary/           Terminos compartidos del ejercicio.
docs/diagrams/           Mermaid (fuente textual), drawio/xml (editable), assets.
docs/cost/               Calculadora de costos LLM y supuestos de tokens.
```

## Por donde empezar

1. [docs/architecture/proposal.md](docs/architecture/proposal.md) - arquitectura maestra.
2. [docs/adr/0005-...md](docs/adr/0005-adopt-chatbot-first-guided-analytics-experience.md) - paradigma chat-first.
3. [docs/adr/0006-...md](docs/adr/0006-adopt-headless-semantic-metrics-layer-and-model-strategy.md) - capa semantica + modelos.
4. [docs/architecture/semantic-layer-and-model-strategy.md](docs/architecture/semantic-layer-and-model-strategy.md) - diseno de capa semantica, entrega de schema y modelos.
5. [docs/adr/0008-...md](docs/adr/0008-adopt-rag-knowledge-layer-and-multi-intent-orchestration.md) - capa de Conocimiento (RAG) + orquestacion multi-intencion.
6. [docs/architecture/rag-knowledge-layer.md](docs/architecture/rag-knowledge-layer.md) - diseno de RAG, ingesta documental y planner que enruta metricas vs. conocimiento.

## Skills

Las skills instaladas se gestionan via `skills-lock.json` y viven en `.agents/skills/`.
`.claude/skills` es un enlace a `.agents/skills/` para que Claude Code las descubra.
