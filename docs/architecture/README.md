# Architecture

Esta carpeta contiene la propuesta de arquitectura de Fase 1.

## Documentos

- [proposal.md](proposal.md): arquitectura Text-to-SQL con Next.js SSR,
  Fastify, Prisma ORM, MCP remoto, PostgreSQL read-only y despliegue
  considerando Cloudflare o Railway.
- [semantic-layer-and-model-strategy.md](semantic-layer-and-model-strategy.md):
  capa semantica de metricas headless y estrategia de modelos (ADR-0006).
- [rag-knowledge-layer.md](rag-knowledge-layer.md): capa de Conocimiento (RAG) y
  orquestacion multi-intencion (ADR-0008); ingesta documental, recuperacion con
  pgvector y planner que enruta metricas vs. conocimiento.
- [../cost/architecture-cost.md](../cost/architecture-cost.md): costo de arquitectura
  por servicio/alojamiento + costo de agentes + costo incremental de RAG.
- [../diagrams/drawio/architecture.drawio](../diagrams/drawio/architecture.drawio):
  diagrama draw.io editable de arquitectura.
- [../diagrams/mermaid/architecture.mmd](../diagrams/mermaid/architecture.mmd):
  fuente Mermaid del diagrama de arquitectura.
- [../diagrams/mermaid/rag-ingestion-flow.mmd](../diagrams/mermaid/rag-ingestion-flow.mmd)
  y
  [../diagrams/mermaid/rag-multi-intent-orchestration.mmd](../diagrams/mermaid/rag-multi-intent-orchestration.mmd):
  fuentes Mermaid de la ingesta RAG y de la orquestacion multi-intencion.

## Pendiente

El diagrama visual de arquitectura se dibujara despues de estabilizar el texto.
Debe incluir web SSR, backend Fastify, Prisma ORM, endpoint MCP, LLM
Orchestrator, SQL Safety Layer, PostgreSQL serverless, auditoria y despliegue.
