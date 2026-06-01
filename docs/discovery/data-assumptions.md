# Data Assumptions

Estas tablas y views no son una implementacion cerrada. Sirven para aterrizar el
requerimiento en una empresa desarrolladora de software.

## Base de Datos

La base principal propuesta es PostgreSQL serverless, preferiblemente Neon Free,
Supabase Free o Railway Postgres. La aplicacion debe usar Prisma ORM con un rol
read-only en runtime y permisos `SELECT` solo sobre views/tablas permitidas.

Se separan credenciales:

- `DATABASE_URL_MIGRATION`: migraciones/setup con permisos controlados.
- `DATABASE_URL_READONLY`: runtime de Fastify/Prisma con permisos solo lectura.

Si el backend vive en Railway, Prisma Client se conecta de forma tradicional a
PostgreSQL. Si vive en Cloudflare Workers, debe validarse Prisma edge/Accelerate
o una estrategia compatible con Workers.

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

## Reportes y Snapshots

Los graficos del dashboard deben alimentarse desde views o snapshots, no desde
preguntas libres de chat. Esto permite SSR estable, menor latencia y menos
dependencia de prompt engineering.

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

Define que graficos aparecen en cada vista SSR.

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
- `question`
- `generated_sql`
- `validated_sql`
- `validation_status`
- `created_at`

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
para respuestas compuestas listas para dashboard.

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

El dashboard ejecutivo no depende de que el usuario escriba prompts. Las
preguntas importantes deben estar abstraidas como reportes, graficos o KPIs.
