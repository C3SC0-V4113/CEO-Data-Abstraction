# Sistema Text-to-SQL Asistido por IA via Web SSR y MCP

Este repositorio documenta la propuesta de arquitectura para un sistema de
consulta de datos asistido por IA. El producto debe permitir que un CEO consulte
los numeros de su empresa en lenguaje natural, vea respuestas ejecutivas,
tablas y graficas, y tambien pueda consumir las mismas capacidades desde un
cliente compatible con MCP como Claude Desktop, Cursor o Codex.

El usuario objetivo se aterriza como el CEO de una empresa desarrolladora de
software. Sus preguntas pueden mezclar indicadores comerciales, financieros y
operativos:

- "Como va el MRR este mes?"
- "Que proyectos estan en riesgo?"
- "Cual es el churn de clientes?"
- "Que margen dejaron los proyectos cerrados?"
- "Puedes darme un forecast de ventas?"
- "Que tickets criticos siguen abiertos?"

La solucion no debe ser solo un chatbot conectado a una base de datos. Debe ser
un sistema Text-to-SQL seguro, auditable y de solo lectura, con una experiencia
web moderna report-first, donde los graficos y reportes resuelven las preguntas
principales sin escribir prompts, y un servidor MCP remoto protegido por
tokens/API keys.

## Arquitectura Objetivo

- **Frontend web**: Next.js fullstack con SSR + shadcn/ui, desplegado en
  Cloudflare Workers usando OpenNext/Cloudflare. La UI principal sera un
  dashboard ejecutivo con reportes, KPIs, tablas y graficas.
- **Login MVP**: un solo usuario CEO creado por seed/setup; no habra registro
  publico ni multiusuario en MVP.
- **Backend principal**: Fastify + TypeScript para APIs, MCP remoto,
  autenticacion, orquestacion LLM, validacion SQL y acceso read-only a datos.
- **Despliegue backend**: Railway recomendado para MVP con Fastify + Prisma;
  Cloudflare Workers como opcion si Prisma edge/Accelerate y MCP funcionan en la
  prueba tecnica.
- **MCP remoto**: endpoint seguro `https://.../mcp`, consumible por Claude
  Desktop, Cursor/Codex u otros clientes compatibles. La web no consume MCP
  directamente.
- **ORM y base de datos**: Prisma ORM sobre PostgreSQL serverless,
  preferiblemente Railway Postgres para MVP.
- **Seguridad**: usuario de base de datos estrictamente read-only, validacion SQL
  por AST, allowlist de views/tablas, `LIMIT`, timeouts, max rows y auditoria.
- **Ingestion MVP**: datos ficticios en PostgreSQL generados por Prisma seed,
  snapshots por reporte y freshness visible en cada vista.
- **Chat contextual**: sidebar o burbuja secundaria; cada grafico/reporte tendra
  boton `Preguntar` para inyectar contexto del reporte.

## LLM Orchestrator

El LLM Orchestrator no es Codex ni Claude. Es un modulo interno dentro del
backend Fastify. Codex y Claude son clientes posibles via MCP.

El orchestrator coordina el flujo completo:

1. Recibe la pregunta en lenguaje natural.
2. Recupera el catalogo de schema, views y metricas permitidas.
3. Construye contexto controlado para el modelo.
4. Genera SQL candidato.
5. Valida el SQL antes de ejecutarlo.
6. Ejecuta con credenciales read-only.
7. Devuelve respuesta ejecutiva, datos, graficas, warnings y `trace_id`.

## Estructura

```text
docs/
  adr/            Decisiones de arquitectura y plantilla ADR.
  discovery/      Problema, usuario CEO, casos de uso, datos asumidos y dudas.
  glossary/       Terminos compartidos del ejercicio.
  strategy/       Opciones de solucion y lineas de brainstorming.
  architecture/   Propuesta de arquitectura y futuro diagrama.
AGENTS.md         Instrucciones para agentes que trabajen en este repo.
```

## Estado Actual

Este repositorio esta en Fase 1: propuesta de arquitectura para aprobacion. No
hay implementacion de producto, base de datos, MCP server ni dashboard todavia.
La prioridad es dejar clara la arquitectura, el flujo Text-to-SQL, las
restricciones de seguridad, las tecnologias y el despliegue.

## Como Leer la Propuesta

1. Empezar por [docs/discovery/problem-framing.md](docs/discovery/problem-framing.md).
2. Revisar los retos de usuarios en [docs/discovery/user-challenges.md](docs/discovery/user-challenges.md).
3. Leer la arquitectura de Fase 1 en [docs/architecture/proposal.md](docs/architecture/proposal.md).
4. Revisar dashboard/reporting en [docs/strategy/dashboard-reporting.md](docs/strategy/dashboard-reporting.md).
5. Revisar MCP en [docs/strategy/mcp-first-access.md](docs/strategy/mcp-first-access.md).
6. Leer la decision de login/despliegue en [docs/adr/0004-adopt-single-ceo-login-and-railway-backend-for-mvp.md](docs/adr/0004-adopt-single-ceo-login-and-railway-backend-for-mvp.md).
7. Revisar diagramas en [docs/diagrams/README.md](docs/diagrams/README.md).
