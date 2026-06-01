# ADR-0002: Adoptar Next.js SSR, Fastify, Prisma y MCP Remoto para Text-to-SQL

## Status

Proposed

## Date

2026-06-01

## Context

El requerimiento actualizado pide un sistema Text-to-SQL con dos frentes:

- Aplicacion web propietaria con chat, tablas y graficas dinamicas.
- Servidor MCP remoto para clientes como Claude Desktop, Cursor/Codex u otros.

El usuario objetivo se aterriza como el CEO de una empresa desarrolladora de
software. El sistema debe consultar datos ejecutivos como MRR, ARR, churn,
pipeline, proyectos, margen, soporte, burn rate, runway y forecast.

Tambien se exige seguridad estricta: autenticacion robusta, conexion de base de
datos read-only y mitigacion contra SQL injection.

## Decision Drivers

- Web app moderna con SSR y experiencia ejecutiva.
- Backend TypeScript adecuado para Fastify, validacion, MCP y orquestacion LLM.
- MCP remoto end-to-end usando la misma logica que la web.
- Base de datos con roles read-only reales.
- Despliegue priorizando Cloudflare para frontend y Railway para backend +
  PostgreSQL en MVP.
- Prisma ORM para modelos tipados, migraciones y una conexion limpia a
  PostgreSQL.
- Seguridad por capas, no solo por prompt.

## Considered Options

### Option 1: Next.js SSR + Fastify + Prisma + PostgreSQL

- Pros: mantiene TypeScript end-to-end, separa frontend y backend, permite
  shadcn/ui, Prisma aporta modelos tipados y PostgreSQL soporta roles read-only
  robustos.
- Cons: dos runtimes y mas piezas de despliegue.

### Option 2: Next.js fullstack solamente

- Pros: menos servicios, TypeScript end-to-end.
- Cons: el servidor MCP y el pipeline Text-to-SQL quedan mas acoplados al
  frontend; la separacion de seguridad queda menos clara.

### Option 3: Fastify + frontend estatico

- Pros: backend simple y frontend barato.
- Cons: no cumple la decision de usar Next.js fullstack SSR.

### Option 4: Cloudflare D1 como base principal

- Pros: muy alineado con Cloudflare y simple para prototipo.
- Cons: no cubre tan bien el requerimiento de permisos `SELECT` estrictos a
  nivel de motor como PostgreSQL.

## Decision

Adoptar:

- Next.js fullstack con SSR + shadcn/ui para la web.
- Fastify + TypeScript como backend principal y servidor MCP remoto.
- Prisma ORM como capa de acceso a PostgreSQL.
- PostgreSQL serverless como base principal.
- Cloudflare Workers como despliegue prioritario para Next.js SSR.
- Railway como opcion recomendada para MVP del backend Fastify + Prisma y
  PostgreSQL.
- Cloudflare Workers como opcion de backend si Prisma edge/Accelerate y el MCP
  SDK funcionan correctamente.

## Rationale

Next.js SSR permite una experiencia web moderna y dinamica para usuarios
ejecutivos. shadcn/ui acelera una interfaz limpia sin disenar un sistema visual
desde cero.

Fastify concentra bien APIs, MCP, validacion, orquestacion LLM y acceso a datos
manteniendo TypeScript end-to-end.
El LLM Orchestrator se define como modulo interno del backend, no como Codex o
Claude. Codex y Claude son clientes MCP.

Prisma ORM se usara para modelos tipados, migraciones y acceso limpio a
PostgreSQL. Para consultas SQL generadas por Text-to-SQL, Prisma solo ejecutara
raw queries despues de pasar por el SQL Safety Layer.

PostgreSQL serverless permite cumplir el requisito de usuario read-only a nivel
de motor. El SQL Safety Layer, Prisma y los permisos de base se complementan: si
falla la validacion de aplicacion, la base aun debe rechazar escrituras.

## Consequences

### Positive

- Cumple web SSR y MCP remoto.
- Reutiliza el mismo pipeline seguro para web y MCP.
- Mantiene el backend preparado para Text-to-SQL, auditoria y data en
  TypeScript.
- Prisma mejora claridad de modelos, migraciones y acceso a PostgreSQL.
- Permite despliegue cercano al free tier usando Cloudflare para frontend y
  Railway para backend + Postgres.

### Negative

- Prisma en Cloudflare Workers requiere validacion edge y puede necesitar Prisma
  Accelerate.
- Hay mas complejidad que una app fullstack unica.
- Requiere disciplina para no duplicar logica entre web y MCP.

### Risks and Mitigations

- Riesgo: Fastify/MCP/Prisma no funciona completo en Cloudflare Workers.
  Mitigacion: desplegar backend en Railway para el MVP y mantener frontend en
  Cloudflare.
- Riesgo: Prisma se usa como unica barrera de seguridad.
  Mitigacion: usar AST validator, allowlists, rol read-only, `LIMIT`, timeout y
  auditoria antes de cualquier raw query.
- Riesgo: SQL generado intenta acciones peligrosas.
  Mitigacion: AST validator, allowlists, read-only role, `LIMIT`, timeout y
  auditoria.
- Riesgo: el CEO recibe respuestas ambiguas.
  Mitigacion: preguntas de aclaracion, definiciones de metricas y warnings.

## Implementation Notes

- El endpoint MCP objetivo sera `POST /mcp` con bearer token.
- La web consumira Fastify para consultas de datos; Next.js route handlers se
  limitaran a UI/session cuando aplique.
- Usar `DATABASE_URL_MIGRATION` para migraciones/setup y `DATABASE_URL_READONLY`
  para runtime.
- El formato comun de respuesta incluira `answer`, `sql`, `data`, `chart`,
  `warnings`, `metadata` y `trace_id`.
- Se priorizara el acceso a views `ceo_*` sobre tablas transaccionales completas.

## Related Decisions

- Supersedes ADR-0001: Adoptar una estrategia MCP-first para acceso a datos con
  IA.

## References

- `docs/architecture/proposal.md`
- `docs/strategy/mcp-first-access.md`
- `docs/discovery/data-assumptions.md`
