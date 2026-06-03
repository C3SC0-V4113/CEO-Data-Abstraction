# Data Assumptions

Estas tablas y views no son una implementacion cerrada. Sirven para aterrizar el
requerimiento en una empresa desarrolladora de software.

## Base de Datos

La base principal propuesta para MVP es PostgreSQL en Railway. La aplicacion
debe usar Prisma ORM con un rol read-only en runtime y permisos `SELECT` solo
sobre views/tablas permitidas.

Se separan credenciales:

- `DATABASE_URL_MIGRATION`: migraciones/setup con permisos controlados.
- `DATABASE_URL_READONLY`: runtime de Fastify/Prisma con permisos solo lectura.

Si el backend vive en Railway, Prisma Client se conecta de forma tradicional a
PostgreSQL. Si vive en Cloudflare Workers, debe validarse Prisma edge/Accelerate
o una estrategia compatible con Workers.

## Auth Seed

El MVP tendra un solo usuario CEO y no tendra registro publico. El usuario se
crea durante seed/setup. La sesion web se maneja con JWT y el rol `CEO` debe
validarse en backend.

### users

- `id`
- `email`
- `password_hash`
- `role`
- `name`
- `created_at`
- `last_login_at`

### sessions

- `id`
- `user_id`
- `token_family_id`
- `expires_at`
- `revoked_at`
- `created_at`

Secrets relacionados:

- `CEO_EMAIL` (backend)
- `CEO_PASSWORD_HASH` (backend)
- `JWT_SECRET` (backend)
- `MCP_API_KEY` (ahora vive en el **servicio MCP**, no en Fastify)
- `CORE_INTERNAL_URL` (servicio MCP -> base de la Core Internal API del backend)
- `CORE_SERVICE_TOKEN` (auth service-to-service entre el servicio MCP y el backend)

## Infraestructura y Borde

El trafico entra por **dos API Gateways Cloudflare** (uno por servicio): Web API Gateway
frente al backend Fastify y MCP API Gateway frente al servicio MCP. Cada uno aplica rate
limiting, throttling, cuotas, WAF/IP rules y limite de tamano (config en Cloudflare). El
**servicio MCP** es un despliegue independiente (Railway) que llama a la Core Internal API
del backend; **no recibe** `DATABASE_URL_*` ni claves del proveedor LLM. Ver ADR-0007.

La **Core Internal API** se protege con defensa en profundidad: la **red privada de Railway**
(`*.railway.internal`) como frontera primaria (sin dominio publico, no enrutada por el Web API
Gateway) + `CORE_SERVICE_TOKEN` validado en cada request. `CORE_INTERNAL_URL` apunta al
hostname interno de Railway. mTLS queda como evolucion futura, fuera del MVP.

## Datos Ficticios y Seed

Para el MVP se trabajara con datos ficticios generados por nosotros mediante
Prisma seed. El seed debe producir datos suficientemente realistas para probar
graficas, filtros, historicos y preguntas contextuales.

El dataset inicial debe cubrir:

- 12 a 18 meses de historico.
- Clientes activos, cancelados y en riesgo.
- Suscripciones con MRR, ARR, expansion y churn.
- Pipeline comercial por etapas.
- Proyectos con presupuesto, costo estimado, revenue reconocido y estado.
- Horas registradas por proyecto.
- Tickets de soporte por prioridad y SLA.
- Gastos por area para burn rate y runway.
- Anomalies intencionales para probar alertas, explicacion de cambios y
  preguntas contextuales.

## Tablas Fuente Candidatas

### customers

Clientes o cuentas.

- `id`
- `name`
- `segment`
- `status`
- `created_at`

### subscriptions

Contratos recurrentes o planes activos.

- `id`
- `customer_id`
- `plan_name`
- `mrr`
- `arr`
- `started_at`
- `ended_at`
- `status`

### invoices

Facturacion emitida.

- `id`
- `customer_id`
- `invoice_date`
- `due_date`
- `status`
- `currency`
- `amount`

### sales_opportunities

Pipeline comercial.

- `id`
- `customer_id`
- `owner_id`
- `stage`
- `expected_close_date`
- `estimated_amount`
- `probability`
- `status`

### projects

Proyectos de delivery.

- `id`
- `customer_id`
- `name`
- `status`
- `start_date`
- `target_end_date`
- `budget`
- `recognized_revenue`
- `estimated_cost`

### time_entries

Horas registradas por proyecto o equipo.

- `id`
- `project_id`
- `employee_id`
- `entry_date`
- `hours`
- `billable`

### support_tickets

Tickets de soporte.

- `id`
- `customer_id`
- `priority`
- `status`
- `sla_due_at`
- `created_at`
- `resolved_at`

### expenses

Costos operativos.

- `id`
- `category`
- `area`
- `expense_date`
- `amount`
- `currency`

## Views Analiticas Recomendadas

El Text-to-SQL debe consultar preferentemente views gobernadas:

- `ceo_revenue_summary`
- `ceo_customer_health`
- `ceo_sales_pipeline`
- `ceo_project_margin`
- `ceo_delivery_risk`
- `ceo_support_health`
- `ceo_financial_runway`

## Artefactos de Chat y Reportes Bajo Demanda

Los reportes del MVP se generan dentro del chat segun la pregunta del CEO. Deben
alimentarse desde views gobernadas y SQL validado, no desde SQL libre. El camino por
defecto es la capa semantica: el LLM produce un `MetricQuery` validado contra el
catalogo de metricas y un generador determinista compila el SQL (ver ADR-0006 y
`docs/architecture/semantic-layer-and-model-strategy.md`). El catalogo de metricas es
configuracion versionada (YAML/JSON), no una tabla de la base. Los snapshots
programados y dashboards persistentes quedan como extension futura.

### conversations

Representa un hilo de chat.

- `id`
- `user_id`
- `title`
- `created_at`
- `updated_at`

### chat_messages

Representa mensajes del usuario y del asistente.

- `id`
- `conversation_id`
- `role`
- `content`
- `intent_mode`
- `extracted_requirements`
- `output_plan`
- `metadata`
- `created_at`

### chat_artifacts

Representa tablas, KPIs, graficas o reportes generados dentro del chat.
`artifact_type` puede ser `text`, `table`, `kpi`, `chart`, `report` o
`action_plan`.

- `id`
- `conversation_id`
- `message_id`
- `artifact_type`
- `title`
- `question`
- `source_views`
- `validated_sql`
- `summary`
- `payload`
- `chart_spec`
- `freshness`
- `warnings`
- `trace_id`
- `created_at`

### chart_edit_messages

Representa el mini chat contextual para editar una grafica existente. Estas
ediciones modifican `chart_spec` sobre datos ya validados. No ejecutan SQL salvo
que se derive una nueva solicitud al chat principal.

- `id`
- `artifact_id`
- `conversation_id`
- `user_id`
- `instruction`
- `previous_chart_spec`
- `updated_chart_spec`
- `change_summary`
- `requires_new_query`
- `warnings`
- `created_at`

## Reportes Persistentes Futuros

Estas tablas pueden usarse si se aprueba despues un historico formal de reportes,
dashboards o automatizaciones. No son la superficie principal del MVP.

### report_definitions

Define reportes disponibles.

- `id`
- `slug`
- `title`
- `category`
- `description`
- `source_view`
- `default_filters`
- `refresh_policy`
- `enabled`

### report_snapshots

Guarda resultados precomputados por periodo y filtros.

- `id`
- `report_id`
- `period_start`
- `period_end`
- `generated_at`
- `ttl_minutes`
- `freshness_status`
- `filters`
- `summary`
- `payload`

### dashboard_widgets

Define que graficos aparecerian en una vista SSR futura.

- `id`
- `route`
- `report_id`
- `widget_type`
- `title`
- `position`
- `default_chart_type`
- `ask_enabled`

## Tablas de Auditoria y Automatizacion

### query_audit_log

Registra cada interaccion web o MCP.

- `id`
- `trace_id`
- `user_id`
- `client_type`
- `path` (`semantic` o `fallback_sql`)
- `question`
- `metric_query`
- `fallback_reason`
- `fallback_schema_context_version`
- `missing_metric_or_dimension`
- `generated_sql`
- `validated_sql`
- `candidate_sql_hash`
- `validated_sql_hash`
- `validation_status`
- `created_at`

Cuando `path = fallback_sql`, el backend tambien debe emitir un log `warn`
`analytics.fallback_sql_triggered` para que desarrollo revise si la pregunta debe
promoverse al catalogo de metricas.

### metric_snapshots

Resultados precomputados para consultas frecuentes o pesadas.

- `id`
- `metric_name`
- `period_start`
- `period_end`
- `dimensions`
- `value`
- `computed_at`

Nota: `metric_snapshots` puede usarse para metricas atomicas; `report_snapshots`
para respuestas compuestas si despues se aprueban reportes persistentes.

### report_jobs

Configuracion de reportes automaticos futuros.

- `id`
- `name`
- `frequency`
- `audience`
- `metric_scope`
- `enabled`

## Supuesto Importante

La IA puede generar SQL candidato, pero solo se ejecuta si pasa validacion por
AST, allowlist y permisos read-only de base de datos.

El chatbot ejecutivo debe guiar al CEO con preguntas sugeridas, acciones rapidas
y aclaraciones. Los reportes se entregan como artefactos dentro de la
conversacion.

La web consume APIs Fastify propias; no consume MCP directamente.
