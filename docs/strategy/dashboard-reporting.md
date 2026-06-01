# Dashboard Reporting

## Principio

La web app sera report-first. Las preguntas ejecutivas mas importantes deben
estar abstraidas como KPIs, tablas y graficas. El chat existe como soporte
global o contextual, pero no debe ser la primera forma de obtener informacion.

## Ingestion de Datos

Para el MVP se usara PostgreSQL con datos ficticios generados por Prisma seed.
No se conectara a sistemas productivos.

El seed debe crear:

- Clientes con estados activos, cancelados y en riesgo.
- Suscripciones con MRR, ARR, expansion y churn.
- Facturas y pagos por periodo.
- Oportunidades comerciales por etapa.
- Proyectos con presupuesto, costo, avance y margen.
- Horas registradas por proyecto.
- Tickets de soporte con prioridad y SLA.
- Gastos por area para burn rate y runway.

## Actualizacion

Los reportes no seran tiempo real en MVP. Se alimentaran por snapshots generados
por seed o job manual/programado.

| Vista | Frecuencia MVP | Motivo |
| --- | --- | --- |
| Overview | Diario o manual | Resumen ejecutivo estable |
| Revenue | Diario | MRR, ARR y crecimiento no requieren segundos |
| Customers | Diario | Churn y riesgo cambian por periodo |
| Pipeline | Diario | Forecast comercial puede revisarse por dia |
| Projects | Diario o cada 6 horas | Riesgo operativo puede cambiar mas rapido |
| Support | Diario o cada 6 horas | Tickets criticos requieren mas frescura |
| Finance | Mensual | Burn rate y runway se leen por cierre |
| Reports history | Por snapshot | Historico de reportes y filtros |

Cada reporte debe mostrar `Ultima actualizacion` y `freshness_status`.

## Vistas SSR

### `/dashboard`

Vista general:

- KPI: MRR actual.
- KPI: churn mensual.
- KPI: runway.
- KPI: proyectos en riesgo.
- KPI: tickets criticos.
- Linea: MRR ultimos 6 meses.

### `/dashboard/revenue`

Revenue:

- Linea: MRR por mes.
- Barras: revenue nuevo vs expansion vs churn.
- Tabla: clientes con mayor cambio de revenue.

### `/dashboard/customers`

Clientes:

- Barras: clientes en riesgo por segmento.
- Tabla: cuentas con churn risk alto.
- KPI: churn rate.
- KPI: expansion revenue.

### `/dashboard/pipeline`

Pipeline:

- Barras: monto por etapa.
- Linea o area: forecast de cierre por mes.
- Tabla: oportunidades principales.

### `/dashboard/projects`

Delivery:

- Barras: margen por proyecto.
- Tabla: proyectos atrasados o sobre presupuesto.
- KPI: margen promedio.
- KPI: proyectos en riesgo.

### `/dashboard/support`

Soporte:

- Barras: tickets por prioridad.
- Linea: tickets abiertos por semana.
- Tabla: clientes con tickets criticos.

### `/dashboard/finance`

Finanzas operativas:

- Linea: burn rate mensual.
- KPI: runway estimado.
- Barras: costos por area.

### `/reports/history`

Historico:

- Tabla de snapshots.
- Filtros por periodo, categoria, freshness y reporte.
- Acceso a reportes anteriores.

## Filtros

Usar filtros solo donde aporten claridad:

- rango de fechas;
- cliente;
- proyecto;
- area/equipo;
- estado;
- severidad/prioridad;
- moneda.

## Boton `Preguntar`

Cada widget debe tener una accion `Preguntar`. Al activarla:

1. Abre sidebar o burbuja de chat.
2. Adjunta contexto del widget.
3. Permite una pregunta focalizada.
4. Responde usando primero el contexto visible.
5. Ejecuta SQL adicional solo si es necesario y seguro.

Contexto minimo:

- `report_id`
- `report_title`
- `active_filters`
- `visible_metrics`
- `chart_type`
- `data_sample`
- `summary`
- `source_view`
- `freshness`

## Endpoints de Reportes

- `GET /api/dashboard/overview`
- `GET /api/reports/revenue`
- `GET /api/reports/customers`
- `GET /api/reports/pipeline`
- `GET /api/reports/projects`
- `GET /api/reports/support`
- `GET /api/reports/finance`
- `POST /api/chat/report-question`

## Criterio de Exito

Un CEO debe poder abrir el dashboard y entender la salud de la empresa sin
escribir un prompt. El chat debe servir para profundizar, explicar o preguntar
sobre un reporte especifico.
