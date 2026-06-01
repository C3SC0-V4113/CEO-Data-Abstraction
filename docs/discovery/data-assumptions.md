# Data Assumptions

Estas tablas no son una implementacion cerrada. Sirven para aterrizar el
brainstorming en un dominio Ventas CRM.

## Tablas Base

### customers

Clientes o cuentas.

- `id`
- `name`
- `segment`
- `region`
- `created_at`

### sellers

Vendedores o ejecutivos comerciales.

- `id`
- `name`
- `team`
- `region`
- `active`

### products

Catalogo de productos o servicios.

- `id`
- `name`
- `category`
- `active`

### sales_orders

Ordenes o ventas cerradas.

- `id`
- `customer_id`
- `seller_id`
- `order_date`
- `status`
- `currency`
- `total_amount`
- `gross_margin`

### sales_order_items

Detalle de productos por venta.

- `id`
- `sales_order_id`
- `product_id`
- `quantity`
- `unit_price`
- `line_total`
- `line_margin`

### sales_targets

Metas comerciales.

- `id`
- `seller_id`
- `period_start`
- `period_end`
- `target_amount`
- `target_margin`

## Tablas de Analitica y Automatizacion

### metric_snapshots

Resultados precomputados para consultas frecuentes o pesadas.

- `id`
- `metric_name`
- `period_start`
- `period_end`
- `dimensions`
- `value`
- `computed_at`

### report_jobs

Configuracion de reportes automaticos.

- `id`
- `name`
- `frequency`
- `audience`
- `metric_scope`
- `enabled`

### report_runs

Ejecuciones historicas de reportes.

- `id`
- `report_job_id`
- `status`
- `started_at`
- `finished_at`
- `summary`
- `artifact_url`

## Datos Documentales para RAG

- Definiciones de metricas.
- Politicas comerciales.
- Reglas de comisiones.
- Criterios de forecast.
- Notas de negocio sobre temporadas, campanas o cambios de precios.

## Supuesto Importante

La IA debe consultar metricas gobernadas, no inventar formulas ni acceder a
tablas internas sin permisos explicitos.
