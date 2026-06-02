# Propuesta de Arquitectura: Text-to-SQL Chatbot Web SSR + MCP

## Objetivo

Construir un sistema de consulta de datos asistido por IA para un CEO de una
empresa desarrolladora de software. El sistema debe aceptar lenguaje natural,
generar SQL seguro de solo lectura, devolver respuestas ejecutivas y renderizar
tablas, KPIs o graficas dinamicas dentro de una experiencia de chatbot.

Debe existir acceso desde dos frentes:

- Web app con login y chatbot ejecutivo.
- Cliente MCP remoto compatible con Claude Desktop, Cursor/Codex u otros.

La web ya no es report-first ni dashboard-first. Para el MVP, las interfaces
propias son solo login y chatbot.

## Stack Propuesto

| Capa | Tecnologia | Despliegue Preferido |
| --- | --- | --- |
| Frontend web | Next.js SSR + shadcn/ui | Cloudflare Workers con OpenNext/Cloudflare |
| Backend API | Fastify + TypeScript | Railway recomendado para MVP |
| MCP remoto | Fastify + MCP TypeScript SDK | Mismo backend Fastify |
| ORM | Prisma ORM | Backend Fastify |
| Base de datos | PostgreSQL | Railway Postgres recomendado para MVP |
| Auth web | JWT + rol `CEO` | Fastify + cookies httpOnly o bearer interno |
| Secretos | Workers Secrets / Railway Variables | Segun capa |
| Reportes futuros | Queues/Workflows/R2 | Cloudflare opcional |

Para un MVP con menor riesgo operativo, el backend Fastify + Prisma y PostgreSQL
se desplegaran en Railway. Cloudflare Workers queda como opcion futura para el
backend si Prisma edge, Prisma Accelerate y el MCP SDK funcionan correctamente
en una prueba tecnica. El frontend SSR se mantiene en Cloudflare Workers.

## Componentes

### Next.js SSR + shadcn/ui

La web app sera una interfaz ejecutiva minimalista chat-first:

- Login obligatorio.
- Chatbot como unica vista autenticada.
- Preguntas sugeridas para CEO.
- Acciones rapidas dentro del chat.
- Respuestas con tablas, KPIs, graficas y narrativa.
- Artefactos de reporte generados bajo demanda dentro del hilo.
- Filtros conversacionales para periodo, cliente, proyecto, area o metrica.
- Freshness, warnings y `trace_id` por respuesta.

Las server actions o route handlers de Next.js se usaran para UI/session cuando
convenga, pero las consultas de datos pasan por Fastify.

La web no consume MCP directamente. MCP es un canal externo para clientes como
Claude Desktop, Cursor/Codex u otros. La web usa APIs HTTP propias de Fastify
para evitar duplicar abstracciones y exponer una superficie innecesaria en el
browser.

Rutas SSR propuestas:

- `/login`
- `/chat`

`/` debe redirigir a `/chat` si existe sesion valida o a `/login` si no existe.

### Fastify + TypeScript

Fastify sera el backend principal:

- `POST /api/auth/login`
- `POST /api/auth/logout`
- `GET /api/auth/session`
- `POST /api/chat/messages`
- `GET /api/chat/conversations`
- `GET /api/chat/conversations/:conversation_id`
- `GET /api/schema/catalog`
- `GET /api/query/history`
- `GET /api/health`
- `POST /mcp`

Fastify concentra autenticacion, JWT, autorizacion por rol, MCP, LLM
Orchestrator, validacion SQL, ejecucion read-only, auditoria y respuesta comun
en TypeScript.

### Authentication and Authorization

El MVP tendra login obligatorio y un solo usuario CEO. No habra registro publico
ni multiusuario operativo.

El usuario CEO se crea por seed/setup. Las credenciales reales no deben quedar
documentadas en texto plano; se configuraran con secretos o hash:

- `CEO_EMAIL`
- `CEO_PASSWORD_HASH`

La sesion web usara JWT:

- access token de vida corta;
- refresh token o sesion persistente si se necesita renovar;
- claim `sub` con `user_id`;
- claim `role` con valor `CEO`;
- claim `session_id` si se conserva tabla de sesiones;
- firma con `JWT_SECRET` o par de llaves segun entorno.

Aunque solo exista el rol `CEO` en el MVP, el backend debe tener middleware de
autorizacion por rol para no acoplar seguridad a una suposicion de UI. CFO, COO
y lideres de area quedan fuera del MVP, pero el modelo no debe impedir agregarlos
despues.

### Data Ingestion and Chat Reporting

Para el MVP se trabajara con datos ficticios creados por seed de PostgreSQL. La
ingestion no representa una integracion productiva todavia; su objetivo es
alimentar respuestas ejecutivas realistas dentro del chatbot.

El flujo sera:

1. Prisma migration crea tablas fuente, views ejecutivas y tablas de auditoria.
2. Prisma seed inserta clientes, suscripciones, facturas, pipeline, proyectos,
   horas, tickets y gastos ficticios.
3. El chatbot consulta views gobernadas cuando el CEO pide un analisis.
4. El backend genera un artefacto de respuesta con narrativa, datos y
   visualizacion.
5. El artefacto queda asociado a la conversacion para preguntas de seguimiento.

Un artefacto de reporte conversacional debe incluir:

- `artifact_id`
- `conversation_id`
- `question`
- `period`
- `source_views`
- `validated_sql`
- `summary`
- `data`
- `chart_spec`
- `freshness`
- `warnings`
- `trace_id`

Los dashboards persistentes, snapshots programados e historico formal de
reportes quedan como extension futura.

### LLM Orchestrator

El LLM Orchestrator es un modulo interno del backend. No es Codex ni Claude.
Codex y Claude son clientes posibles via MCP.

Responsabilidades:

- Recibir pregunta en lenguaje natural.
- Cargar rol, permisos, schema y views permitidas.
- Cargar contexto conversacional minimo.
- Construir contexto para el LLM.
- Pedir aclaraciones si la pregunta es ambigua.
- Generar SQL candidato cuando se requieran datos.
- Enviar SQL al SQL Safety Layer.
- Ejecutar SQL validado.
- Generar respuesta ejecutiva.
- Proponer visualizacion.
- Persistir auditoria y artefactos conversacionales.

### SQL Safety Layer

Valida el SQL antes de ejecutarlo:

- Solo permite `SELECT`.
- Bloquea DDL, DML y multiples statements.
- Usa AST parser, no solo regex.
- Aplica allowlist de views, tablas, columnas y funciones.
- Aplica permisos segun rol.
- Fuerza `LIMIT`, max rows y timeout.
- Rechaza tablas internas no autorizadas.

### Database Layer

La base principal sera PostgreSQL serverless. Prisma ORM sera la capa de acceso
a datos para modelos tipados, migraciones y conexion limpia a PostgreSQL.

Se usaran dos credenciales:

- `DATABASE_URL_MIGRATION`: solo para migraciones/setup en entorno controlado.
- `DATABASE_URL_READONLY`: usada por Fastify/Prisma en runtime.

El usuario runtime debe ser read-only con permisos `SELECT` solo sobre views
analiticas o tablas permitidas. Railway Postgres sera la opcion favorita para el
MVP por compatibilidad directa con Prisma Client y el backend Fastify.

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

Las tools MCP usan los mismos servicios internos del chatbot, pero MCP no
reemplaza las APIs web. La web SSR consume `/api/auth/*`, `/api/chat/*` y
`/api/schema/*`; MCP queda expuesto solo para clientes externos compatibles.

## Flujo de Datos

### Flujo Login y JWT

1. El CEO abre `/login`.
2. Next.js envia credenciales a `POST /api/auth/login`.
3. Fastify valida hash de password del usuario seeded.
4. Fastify emite JWT con rol `CEO`.
5. Next.js guarda la sesion en cookie httpOnly o mecanismo equivalente.
6. El CEO entra a `/chat`.

### Flujo Chat-First

1. El CEO escribe una pregunta o selecciona una sugerencia.
2. Next.js envia mensaje a `POST /api/chat/messages`.
3. Fastify valida JWT, rol y sesion.
4. El LLM Orchestrator carga contexto permitido y estado conversacional.
5. Si falta informacion, el sistema pide aclaracion.
6. Si requiere datos, el LLM genera SQL candidato.
7. SQL Safety Layer valida por AST, allowlists, rol y limites.
8. Database Layer ejecuta con usuario read-only.
9. El backend genera narrativa, tabla/grafica/KPI y warnings.
10. Fastify registra auditoria y devuelve `trace_id`.
11. Next.js renderiza el artefacto dentro del chat.

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
- Historial conversacional minimo.
- Ultimo artefacto relevante, si aplica.

No debe incluir secretos, credenciales, tablas bloqueadas ni datos sensibles
fuera del alcance.

## Seguridad

- Autenticacion obligatoria para web y MCP.
- JWT para sesion web.
- Bearer token/API key para MCP remoto.
- Login web con usuario CEO seeded; sin registro publico.
- Middleware de autorizacion por rol.
- Rol PostgreSQL read-only.
- Prisma Client usando `DATABASE_URL_READONLY` en runtime.
- Migraciones con `DATABASE_URL_MIGRATION` fuera del runtime de consulta.
- SQL Safety Layer antes de ejecutar.
- Allowlist de schema.
- `LIMIT`, timeout y max rows obligatorios.
- Auditoria de prompts, SQL generado, SQL validado, cliente y usuario.
- No exponer schema sin autenticacion.
- Separar tokens MCP de credenciales DB y LLM.
- Separar auth web de MCP; el frontend no debe usar `MCP_API_KEY`.
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

Para artefactos generados en chat se extiende con:

```json
{
  "artifact_id": "artifact_uuid",
  "conversation_id": "conversation_uuid",
  "question": "Mostrar MRR y crecimiento de los ultimos 6 meses",
  "period": "last_6_months",
  "source_views": ["ceo_revenue_summary"],
  "freshness": {
    "generated_at": "2026-06-01T08:00:00Z",
    "status": "fresh"
  },
  "summary": "MRR crecio 8% en los ultimos 6 meses.",
  "chart_spec": {
    "type": "line",
    "x": "month",
    "y": "mrr"
  }
}
```

## Actualizacion de Reportes

El MVP no promete tiempo real ni reportes programados. Los datos se alimentan por
seed y se consultan bajo demanda desde el chatbot. Si una consulta es costosa o
frecuente, puede introducirse cache o snapshot conversacional, pero no como
dashboard principal.

## Entregables de Fase 2

Cuando se apruebe la arquitectura y se desarrolle la solucion, los entregables
seran:

- URL de aplicacion web.
- URL del servidor MCP.
- API key o token valido para MCP.
- Instrucciones de configuracion para cliente MCP.
- Credenciales de prueba o usuario demo para staging.
