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

La web app sera una interfaz ejecutiva minimalista:

- Ventana de chat.
- Panel de resultados.
- Tablas.
- Graficas de linea, barras, area y KPIs.
- Preguntas sugeridas para CEO.

Las server actions o route handlers de Next.js se usaran para UI/session cuando
convenga, pero las consultas de datos pasan por Fastify.

### Fastify + TypeScript

Fastify sera el backend principal:

- `POST /api/chat/query`
- `GET /api/schema/catalog`
- `GET /api/query/history`
- `GET /api/health`
- `POST /mcp`

Fastify concentra autenticacion, MCP, LLM Orchestrator, validacion SQL,
ejecucion read-only, auditoria y respuesta comun en TypeScript.

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

## Flujo de Datos

1. El CEO escribe una pregunta en la web o en un cliente MCP.
2. El cliente envia la solicitud a Fastify.
3. Fastify autentica al usuario o API key.
4. El LLM Orchestrator carga schema permitido y definiciones.
5. El LLM genera SQL candidato.
6. SQL Safety Layer valida el SQL.
7. Database Layer ejecuta el SQL con usuario read-only.
8. Visualization Layer propone tabla, KPI o grafica.
9. Fastify registra auditoria y devuelve `trace_id`.
10. Next.js o el cliente MCP muestra respuesta, datos y warnings.

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

## Entregables de Fase 2

Cuando se apruebe la arquitectura y se desarrolle la solucion, los entregables
seran:

- URL de aplicacion web.
- URL del servidor MCP.
- API key o token valido.
- Instrucciones de configuracion para cliente MCP.
- Credenciales de prueba o usuario demo para staging.
