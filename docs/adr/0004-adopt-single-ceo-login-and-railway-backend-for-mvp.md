# ADR-0004: Adoptar Login Unico CEO y Railway Backend para MVP

## Status

Proposed

## Date

2026-06-01

## Context

El MVP necesita reducir alcance y complejidad. La aplicacion estara orientada a
un CEO de una empresa desarrolladora de software. La experiencia vigente sera
chat-first: despues del login, el CEO usa una unica vista de chatbot para pedir
analisis y generar reportes bajo demanda dentro de la conversacion.

Tambien se busca reducir friccion de despliegue para Fastify + Prisma +
PostgreSQL. Aunque el MVP tenga un solo usuario operativo, la autenticacion debe
usar JWT y la autorizacion debe validar rol en backend.

## Decision Drivers

- Evitar construir registro, multiusuario y RBAC completo en MVP.
- Mantener JWT y middleware de autorizacion por rol desde el inicio.
- Mantener compatibilidad directa entre Fastify, Prisma Client y PostgreSQL.
- Separar frontend SSR de backend/API.
- Evitar que el frontend consuma MCP directamente.
- Mantener MCP como canal externo seguro para clientes compatibles.

## Considered Options

### Option 1: Login unico CEO con JWT + Railway backend

- Pros: menor complejidad, Prisma funciona de forma tradicional, backend y
  Postgres conviven en Railway, auth es simple pero no informal.
- Cons: no cubre multiusuario operativo ni roles adicionales.

### Option 2: Registro y multiusuario desde MVP

- Pros: mas cercano a producto SaaS.
- Cons: incrementa alcance, RBAC, gestion de usuarios y pruebas.

### Option 3: Backend en Cloudflare Workers desde MVP

- Pros: mas Cloudflare-native.
- Cons: Prisma edge/Accelerate y MCP SDK requieren prueba tecnica adicional.

### Option 4: Frontend consume MCP directamente

- Pros: una abstraccion unica para IA.
- Cons: expone superficie innecesaria al browser, mezcla auth web con MCP auth y
  duplica patrones que Fastify ya puede resolver con HTTP APIs propias.

## Decision

Adoptar para el MVP:

- Login obligatorio sin registro.
- Un solo usuario CEO creado por seed/setup.
- Sesion web con JWT.
- Claim de rol `CEO` en el token.
- Middleware backend de autorizacion por rol.
- Backend Fastify + Prisma en Railway.
- PostgreSQL en Railway o servicio compatible dentro del mismo entorno.
- Next.js SSR + shadcn/ui en Cloudflare Workers.
- MCP como canal externo protegido con `MCP_API_KEY`.
- Frontend no consume MCP directamente; consume APIs Fastify.

## Rationale

Railway reduce riesgo para Fastify + Prisma + PostgreSQL porque usa un entorno
Node.js tradicional. Esto evita resolver edge constraints antes de validar la
propuesta.

Un usuario CEO seeded permite demostrar el flujo completo sin construir registro
ni multiusuario. Aun asi, JWT y autorizacion por rol evitan que la seguridad
dependa de la UI o de una suposicion informal. MCP queda disponible para
clientes externos sin mezclar su token con el frontend.

## Consequences

### Positive

- Menor alcance para MVP.
- Despliegue backend mas compatible con Prisma.
- Seguridad web modelada con JWT y rol desde el inicio.
- Separacion clara entre web auth y MCP auth.
- Frontend SSR mantiene contratos HTTP simples.

### Negative

- No hay autoservicio de usuarios.
- No hay roles CFO/COO/lideres en MVP.
- Hay que implementar emision, validacion y expiracion de JWT aunque solo exista
  un usuario.
- Railway agrega otro proveedor junto a Cloudflare.

### Risks and Mitigations

- Riesgo: credenciales CEO se filtran.
  Mitigacion: usar hash/secrets y no documentar valores reales.
- Riesgo: MVP no representa multiusuario real.
  Mitigacion: documentar multiusuario como futuro, pero mantener claims y
  middleware de rol.
- Riesgo: confusion entre web API y MCP.
  Mitigacion: documentar que MCP es externo y el frontend no lo consume.
- Riesgo: JWT se valida solo en frontend.
  Mitigacion: validar JWT y rol en cada endpoint protegido de Fastify.

## Implementation Notes

- Seed crea usuario CEO con `CEO_EMAIL` y `CEO_PASSWORD_HASH`.
- JWT usa `JWT_SECRET` o par de llaves configurado por secreto.
- JWT incluye `sub`, `role`, `exp` y opcionalmente `session_id`.
- Usar `DATABASE_URL_MIGRATION` para migraciones/setup.
- Usar `DATABASE_URL_READONLY` para runtime de consultas.
- MCP usa `MCP_API_KEY` en `Authorization: Bearer <token>`.
- Web usa `/api/auth/*`, `/api/chat/*` y `/api/schema/*`, no `/mcp`.

## Related Decisions

- ADR-0002: Adoptar Next.js SSR, Fastify, Prisma y MCP remoto para Text-to-SQL.
- ADR-0005: Adoptar chatbot ejecutivo como experiencia principal.

## References

- `docs/architecture/proposal.md`
- `docs/discovery/open-questions.md`
- `docs/strategy/mcp-first-access.md`
