# Propuesta de Arquitectura: Text-to-SQL Web SSR + MCP

## Objetivo

Construir un sistema de consulta de datos asistido por IA para un CEO de una
empresa desarrolladora de software. El sistema debe aceptar lenguaje natural,
generar SQL seguro de solo lectura, devolver respuestas ejecutivas y renderizar
tablas o graficas dinamicas.

Debe existir acceso desde dos frentes:

- Web app fullstack con SSR.
- Cliente MCP remoto compatible con Claude Desktop, Cursor/Codex u otros.

## Stack Propuesto

| Capa | Tecnologia | Despliegue Preferido |
| --- | --- | --- |
| Frontend web | Next.js SSR + shadcn/ui | Cloudflare Workers con OpenNext/Cloudflare |
| Backend API | Fastify + TypeScript | Railway recomendado para MVP; Cloudflare Workers como opcion |
| MCP remoto | Fastify + MCP TypeScript SDK | Mismo backend Fastify |
| ORM | Prisma ORM | Backend Fastify |
| Base de datos | PostgreSQL serverless | Neon Free, Supabase Free o Railway Postgres |
| Conexion DB desde Cloudflare | Prisma Accelerate o Prisma Postgres, segun prueba tecnica | Cloudflare |
| Secretos | Workers Secrets | Cloudflare |
| Reportes futuros | Queues/Workflows/R2 | Cloudflare opcional |

Para un MVP con menor riesgo operativo, el backend Fastify + Prisma se
desplegara en Railway. Cloudflare Workers queda como opcion si Prisma edge,
Prisma Accelerate y el MCP SDK funcionan correctamente en la prueba tecnica.

## Componentes

### Next.js SSR + shadcn/ui

La web app sera una interfaz ejecutiva minimalista report-first:

- Dashboard ejecutivo como vista principal.
- Reportes y graficas predefinidas.
- Tablas.
- Graficas de linea, barras, area y KPIs.
- Preguntas sugeridas para CEO.
- Sidebar o burbuja de chat como soporte secundario.
- Boton `Preguntar` en cada reporte/grafico para abrir chat contextual.

Las server actions o route handlers de Next.js se usaran para UI/session cuando
convenga, pero las consultas de datos pasan por Fastify.

Rutas SSR propuestas:

- `/dashboard`
- `/dashboard/revenue`
- `/dashboard/customers`
- `/dashboard/pipeline`
- `/dashboard/projects`
- `/dashboard/support`
- `/dashboard/finance`
- `/reports/history`

### Fastify + TypeScript

Fastify sera el backend principal:

- `POST /api/chat/query`
- `POST /api/chat/report-question`
- `GET /api/dashboard/overview`
- `GET /api/reports/revenue`
- `GET /api/reports/customers`
- `GET /api/reports/pipeline`
- `GET /api/reports/projects`
- `GET /api/reports/support`
- `GET /api/reports/finance`
- `GET /api/schema/catalog`
- `GET /api/query/history`
- `GET /api/health`
- `POST /mcp`

Fastify concentra autenticacion, MCP, LLM Orchestrator, validacion SQL,
ejecucion read-only, auditoria y respuesta comun en TypeScript.

### Data Ingestion and Reporting

Para el MVP se trabajara con datos ficticios creados por seed de PostgreSQL. La
ingestion no representa una integracion productiva todavia; su objetivo es
alimentar reportes ejecutivos realistas.

El flujo sera:

1. Prisma migration crea tablas fuente, views ejecutivas y tablas de snapshots.
2. Prisma seed inserta clientes, suscripciones, facturas, pipeline, proyectos,
   horas, tickets y gastos ficticios.
3. Un job manual o script de seed genera snapshots por reporte.
4. Las rutas SSR consumen snapshots o views ya agregadas.
5. El chat contextual puede usar el snapshot visible antes de pedir SQL nuevo.

Los snapshots deben incluir:

- `report_id`
- `period_start`
- `period_end`
- `generated_at`
- `ttl_minutes`
- `freshness_status`
- `filters`
- `summary`
- `payload`

### LLM Orchestrator

El LLM Orchestrator es un modulo interno del backend. No es Codex ni Claude.
Codex y Claude son clientes posibles via MCP.

Responsabilidades:

- Recibir pregunta en lenguaje natural.
- Cargar schema y views permitidas.
- Construir contexto para el LLM.
- Generar SQL candidato.
- Enviar SQL al SQL Safety Layer.
- Ejecutar SQL validado.
- Generar respuesta ejecutiva.
- Proponer visualizacion.
- Responder preguntas globales o focalizadas con contexto de reporte.

### SQL Safety Layer

Valida el SQL antes de ejecutarlo:

- Solo permite `SELECT`.
- Bloquea DDL, DML y multiples statements.
- Usa AST parser, no solo regex.
- Aplica allowlist de views, tablas, columnas y funciones.
- Fuerza `LIMIT`, max rows y timeout.
- Rechaza tablas internas no autorizadas.

### Database Layer

La base principal sera PostgreSQL serverless. Prisma ORM sera la capa de acceso
a datos para modelos tipados, migraciones y conexion limpia a PostgreSQL.

Se usaran dos credenciales:

- `DATABASE_URL_MIGRATION`: solo para migraciones/setup en entorno controlado.
- `DATABASE_URL_READONLY`: usada por Fastify/Prisma en runtime.

El usuario runtime debe ser read-only con permisos `SELECT` solo sobre views
analiticas o tablas permitidas.

Views sugeridas:

- `ceo_revenue_summary`
- `ceo_customer_health`
- `ceo_sales_pipeline`
- `ceo_project_margin`
- `ceo_delivery_risk`
- `ceo_support_health`
- `ceo_financial_runway`

### MCP Remote Server

El endpoint MCP remoto debe estar protegido:

```text
POST https://<host>/mcp
Authorization: Bearer <token>
```

Tools candidatas:

- `describe_business_schema`
- `ask_company_data`
- `run_readonly_query`
- `suggest_executive_questions`
- `generate_chart_spec`

Las tools MCP pueden consultar reportes existentes, pero la experiencia web debe
priorizar graficos predefinidos antes que chat libre.

## Flujo de Datos

### Flujo Report-First

1. El CEO entra a una vista SSR de dashboard.
2. Next.js solicita datos a Fastify para el reporte correspondiente.
3. Fastify lee snapshots, views ejecutivas o queries Prisma permitidas.
4. Next.js renderiza KPIs, tablas y graficas.
5. Cada reporte muestra freshness y boton `Preguntar`.

### Flujo de Chat Contextual

1. El CEO presiona `Preguntar` en un reporte o grafico.
2. Next.js abre sidebar o burbuja de chat.
3. La pregunta se envia a `POST /api/chat/report-question`.
4. El payload incluye `report_id`, filtros activos, metricas visibles,
   `context_summary`, `source_view`, muestra de datos y freshness.
5. El LLM Orchestrator responde usando primero el contexto visible.
6. Si necesita datos extra, genera SQL candidato y pasa por SQL Safety Layer.
7. Fastify registra auditoria y devuelve `trace_id`.

### Flujo MCP

1. El CEO o usuario tecnico pregunta desde un cliente MCP.
2. Fastify valida token y scope.
3. El LLM Orchestrator carga schema permitido y definiciones.
4. El LLM genera SQL candidato si hace falta.
5. SQL Safety Layer valida el SQL.
6. Database Layer ejecuta con usuario read-only.
7. Fastify registra auditoria y devuelve narrativa, datos y warnings.

## Manejo del Contexto LLM

El contexto enviado al LLM debe incluir solo lo necesario:

- Rol del usuario.
- Views y columnas permitidas.
- Definiciones de metricas.
- Reglas de formato SQL.
- Limites de seguridad.
- Historial conversacional minimo si aplica.

No debe incluir secretos, credenciales, tablas bloqueadas ni datos sensibles
fuera del alcance.

## Seguridad

- Autenticacion obligatoria para web y MCP.
- Bearer token/API key para MCP remoto.
- Rol PostgreSQL read-only.
- Prisma Client usando `DATABASE_URL_READONLY` en runtime.
- Migraciones con `DATABASE_URL_MIGRATION` fuera del runtime de consulta.
- SQL Safety Layer antes de ejecutar.
- Allowlist de schema.
- `LIMIT`, timeout y max rows obligatorios.
- Auditoria de prompts, SQL generado, SQL validado, cliente y usuario.
- No exponer schema sin autenticacion.
- Separar tokens MCP de credenciales DB y LLM.
- No usar Prisma como unica barrera de seguridad; las raw queries generadas por
  IA solo pueden ejecutarse despues de validacion AST.

## Respuesta Comun

La web y MCP deben compartir una estructura:

```json
{
  "answer": "Resumen ejecutivo en lenguaje natural.",
  "sql": "SELECT ...",
  "data": [],
  "chart": {
    "type": "line",
    "x": "month",
    "y": "mrr"
  },
  "warnings": [],
  "metadata": {
    "client_type": "web",
    "rows": 12
  },
  "trace_id": "uuid"
}
```

Para reportes y chat contextual se extiende con:

```json
{
  "report_id": "revenue_mrr_trend",
  "report_title": "MRR y crecimiento",
  "filters": {
    "period": "last_6_months"
  },
  "freshness": {
    "generated_at": "2026-06-01T08:00:00Z",
    "ttl_minutes": 1440,
    "status": "fresh"
  },
  "context_summary": "MRR crecio 8% en los ultimos 6 meses.",
  "source_view": "ceo_revenue_summary",
  "chart_specs": []
}
```

## Actualizacion de Reportes

El MVP no promete tiempo real. Los reportes se alimentan por seed y snapshots:

- Overview: diario o manual.
- Revenue, customers y pipeline: diario.
- Projects y support: diario o cada 6 horas si se simula mayor dinamismo.
- Finance: mensual.
- Reports history: conserva snapshots por periodo.

## Entregables de Fase 2

Cuando se apruebe la arquitectura y se desarrolle la solucion, los entregables
seran:

- URL de aplicacion web.
- URL del servidor MCP.
- API key o token valido.
- Instrucciones de configuracion para cliente MCP.
- Credenciales de prueba o usuario demo para staging.
