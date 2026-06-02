# ADR-0003: Adoptar Dashboard Report-First con Chat Contextual

## Status

Superseded

## Date

2026-06-01

## Context

El requerimiento pide una web app con ventana de chat y area de renderizado para
componentes dinamicos como tablas y graficas. La interpretacion inicial podia
llevar a una experiencia centrada en chat, pero el objetivo de negocio es que el
CEO obtenga respuestas importantes sin escribir prompts.

Para el MVP se trabajara con datos ficticios creados por seed de PostgreSQL. Las
preguntas importantes deben abstraerse como reportes, KPIs y graficas. El chat
queda como una capa secundaria para preguntas globales o focalizadas sobre un
reporte.

## Decision Drivers

- Reducir dependencia de prompt engineering.
- Aprovechar Next.js SSR para vistas separadas y cargadas desde servidor.
- Alimentar graficas desde endpoints y snapshots, no desde chat libre.
- Permitir preguntas especificas sobre un reporte con contexto ya preparado.
- Mantener seguridad Text-to-SQL con SQL Safety Layer y rol read-only.

## Considered Options

### Option 1: Chat-first

- Pros: simple de demostrar y flexible.
- Cons: obliga al CEO a saber que preguntar; desaprovecha reportes
  predefinidos.

### Option 2: Dashboard report-first con chat contextual

- Pros: resuelve preguntas frecuentes sin prompt, permite profundizar por
  reporte, encaja con SSR y reduce ambiguedad.
- Cons: requiere disenar reportes, snapshots, vistas y contratos de contexto.

### Option 3: Dashboard sin chat

- Pros: interfaz mas simple.
- Cons: pierde capacidad de explicar y explorar casos especificos.

## Decision

Adoptar una experiencia report-first:

- Dashboard ejecutivo como vista principal.
- Reportes separados por dominio: overview, revenue, customers, pipeline,
  projects, support, finance e historico.
- Chat global como sidebar/burbuja secundaria.
- Boton `Preguntar` por reporte/grafico para abrir chat contextual.
- Datos iniciales por seed y snapshots, no tiempo real.

## Rationale

El dashboard responde las preguntas importantes sin depender de prompts. El chat
contextual conserva valor porque permite preguntar sobre un grafico especifico
sin que el usuario tenga que reconstruir contexto manualmente.

Los snapshots hacen que SSR sea estable, predecible y rapido para el MVP.
Tiempo real no es necesario para validar el requerimiento.

## Consequences

### Positive

- Menos friccion para el CEO.
- Mejor uso de graficas y tablas dinamicas.
- Preguntas focalizadas con mas contexto y menos ambiguedad.
- Separacion clara entre reportes predefinidos y exploracion con IA.

### Negative

- Mayor trabajo de modelado de reportes.
- Hay que mantener freshness y snapshots.
- El chat libre queda subordinado a los reportes.

### Risks and Mitigations

- Riesgo: reportes iniciales no cubren preguntas importantes.
  Mitigacion: historico de preguntas y feedback para priorizar nuevos widgets.
- Riesgo: snapshots desactualizados confunden al usuario.
  Mitigacion: mostrar `Ultima actualizacion` y `freshness_status`.
- Riesgo: chat contextual ignore el reporte visible.
  Mitigacion: enviar `report_id`, filtros, muestra de datos, summary y
  `source_view` en cada pregunta contextual.

## Implementation Notes

- Crear `report_definitions`, `report_snapshots` y `dashboard_widgets`.
- Agregar endpoints `GET /api/reports/*` y `POST /api/chat/report-question`.
- Cada widget debe incluir accion `Preguntar`.
- Para MVP, generar datos y snapshots con Prisma seed o job manual.

## Related Decisions

- ADR-0002: Adoptar Next.js SSR, Fastify, Prisma y MCP remoto para Text-to-SQL.
- ADR-0005: Adoptar chatbot ejecutivo como experiencia principal.

## References

- `docs/architecture/proposal.md`
- `docs/strategy/dashboard-reporting.md`
- `docs/discovery/data-assumptions.md`
