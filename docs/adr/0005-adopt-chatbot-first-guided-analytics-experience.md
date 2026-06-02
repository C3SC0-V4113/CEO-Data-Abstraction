# ADR-0005: Adoptar Chatbot Ejecutivo como Experiencia Principal

## Status

Proposed

## Date

2026-06-01

## Context

La estrategia anterior proponia una aplicacion report-first: dashboard ejecutivo,
vistas SSR de reportes y chat contextual secundario. El nuevo paradigma cambia
la experiencia principal: la web debe comportarse como una aplicacion estilo
chatbot. El CEO no navega primero por dashboards ni por un historico de reportes;
entra a una conversacion y pide el analisis que necesita.

La autenticacion sigue siendo para un solo CEO en MVP, pero debe manejarse con
JWT y seguridad basada en roles. El rol inicial es `CEO`; no hay registro publico
ni multiusuario operativo, pero el backend no debe depender de que la UI sea la
unica barrera de seguridad.

## Decision Drivers

- Reducir la superficie de interfaz del MVP a login y chatbot.
- Permitir que los reportes se generen bajo demanda dentro de la conversacion.
- Evitar construir dashboards, rutas de reportes e historico antes de validar el
  flujo conversacional.
- Mantener seguridad con JWT, autorizacion por rol, SQL Safety Layer y
  PostgreSQL read-only.
- Conservar MCP remoto como canal externo sobre el mismo pipeline.

## Considered Options

### Option 1: Dashboard report-first con chat contextual

- Pros: reduce el esfuerzo de formular preguntas desde cero y muestra KPIs
  frecuentes al abrir la app.
- Cons: contradice el nuevo enfoque chat-first, aumenta alcance de UI y
  subordina el chatbot.

### Option 2: Chatbot first con reportes generados dentro del chat

- Pros: alinea la experiencia con el nuevo paradigma, reduce pantallas,
  permite analisis bajo demanda y conserva continuidad conversacional.
- Cons: exige buenas preguntas sugeridas, manejo riguroso de contexto y
  artefactos visuales dentro del hilo.

### Option 3: Chat libre sin guia

- Pros: interfaz minima y flexible.
- Cons: obliga al CEO a saber que preguntar y aumenta riesgo de ambiguedad.

## Decision

Adoptar una experiencia chat-first:

- La web tendra solo login y chatbot como interfaces principales.
- Los reportes se generan dentro de la vista de chat segun la pregunta del CEO.
- Las respuestas pueden incluir narrativa, KPIs, tablas, graficas y warnings.
- El chatbot debe sugerir preguntas y pedir aclaraciones cuando la solicitud sea
  ambigua.
- El backend usara JWT con claim de rol y middleware de autorizacion.
- El rol inicial sera `CEO`, creado por seed/setup.
- No habra dashboard, rutas de reportes ni historico visual como superficie MVP.

## Rationale

El nuevo objetivo privilegia una experiencia conversacional. Mantener un
dashboard como pantalla principal forzaria un modelo mental anterior y desviaria
esfuerzo hacia widgets, rutas y filtros que ya no son el centro del producto.

El chatbot first no significa SQL libre ni ausencia de gobernanza. La seguridad
sigue dependiendo del SQL Safety Layer, PostgreSQL read-only, allowlists,
auditoria y autorizacion por rol. Las preguntas sugeridas y acciones rapidas
reducen el problema de prompt engineering sin volver a un dashboard.

## Consequences

### Positive

- Menor superficie de UI para el MVP.
- Experiencia alineada con el nuevo paradigma.
- Reportes dinamicos segun necesidad real del CEO.
- Mejor continuidad para preguntas de seguimiento.
- Seguridad web modelada desde el inicio con JWT y rol.

### Negative

- Se pierde la visibilidad inmediata de KPIs al abrir la app.
- El valor inicial depende de buenas sugerencias y ejemplos.
- El manejo de contexto conversacional se vuelve critico.
- La generacion de artefactos visuales dentro del chat requiere contratos claros.

### Risks and Mitigations

- Riesgo: el CEO no sabe que preguntar al abrir un chat vacio.
  Mitigacion: mostrar preguntas sugeridas, acciones rapidas y ejemplos por
  dominio.
- Riesgo: el sistema mezcla filtros de preguntas anteriores.
  Mitigacion: persistir estado conversacional explicito y mostrar filtros
  activos en cada artefacto.
- Riesgo: una respuesta generada parece reporte formal sin suficiente contexto.
  Mitigacion: incluir freshness, warnings, source views y `trace_id`.
- Riesgo: el rol `CEO` se asume solo en frontend.
  Mitigacion: validar JWT y rol en cada endpoint Fastify.

## Implementation Notes

- Rutas web principales: `/login` y `/chat`.
- Endpoints web principales: `/api/auth/*`, `/api/chat/*` y
  `/api/schema/catalog`.
- `POST /api/chat/messages` debe devolver narrativa, datos, `chart_spec`,
  warnings y `trace_id`.
- Los artefactos de reporte conversacional deben asociarse a `conversation_id`.
- JWT debe incluir `sub`, `role` y expiracion.
- El middleware de autorizacion debe validar rol antes de exponer tools, schema o
  consultas.

## Related Decisions

- ADR-0002: Adoptar Next.js SSR, Fastify, Prisma y MCP remoto para Text-to-SQL.
- ADR-0003: Adoptar dashboard report-first con chat contextual. Superseded por
  este ADR.
- ADR-0004: Adoptar login unico CEO y Railway backend para MVP.

## References

- `docs/strategy/guided-analytics-experience.md`
- `docs/architecture/proposal.md`
- `docs/discovery/open-questions.md`
