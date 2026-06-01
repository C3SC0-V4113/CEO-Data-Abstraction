# Architecture Decision Records

Este directorio contiene Architecture Decision Records (ADRs) para la propuesta
"Hablar con la Data sin Depender Siempre del Chat".

## Indice

| ADR | Titulo | Estado | Fecha |
| --- | --- | --- | --- |
| [0001](0001-adopt-mcp-first-data-access-strategy.md) | Adoptar una estrategia MCP-first para acceso a datos con IA | Superseded | 2026-06-01 |
| [0002](0002-adopt-nextjs-ssr-fastify-prisma-mcp-text-to-sql.md) | Adoptar Next.js SSR, Fastify, Prisma y MCP remoto para Text-to-SQL | Proposed | 2026-06-01 |
| [0003](0003-adopt-report-first-dashboard-with-contextual-chat.md) | Adoptar dashboard report-first con chat contextual | Proposed | 2026-06-01 |
| [0004](0004-adopt-single-ceo-login-and-railway-backend-for-mvp.md) | Adoptar login unico CEO y Railway backend para MVP | Proposed | 2026-06-01 |

## Crear un Nuevo ADR

1. Copiar `template.md` a `NNNN-titulo-con-guiones.md`.
2. Completar contexto, drivers, opciones, decision y consecuencias.
3. Mantener el estado en `Proposed` hasta que la decision se acepte.
4. Actualizar este indice.

## Estados

- `Proposed`: en discusion.
- `Accepted`: decision aprobada.
- `Rejected`: evaluada y descartada.
- `Deprecated`: ya no se recomienda.
- `Superseded`: reemplazada por otro ADR.
