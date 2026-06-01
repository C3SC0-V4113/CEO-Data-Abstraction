# Use Cases

## Revenue Ejecutivo

El CEO pregunta por MRR, ARR, ventas mensuales, crecimiento contra el mes
anterior y forecast. La respuesta debe incluir cifras, variacion, tendencia y
una grafica de linea o KPI segun corresponda.

## Churn y Clientes en Riesgo

El CEO quiere entender perdida de clientes, expansion revenue, cuentas en riesgo
y clientes con senales de deterioro. La respuesta debe explicar la metrica usada
y listar clientes relevantes si el rol tiene permiso.

## Pipeline Comercial

El CEO pregunta por pipeline, oportunidades por etapa, conversion y forecast de
cierre. La visualizacion sugerida puede ser barras por etapa o tabla de
oportunidades principales.

## Proyectos en Riesgo

El CEO quiere saber que proyectos estan atrasados, fuera de presupuesto o con
margen bajo. La respuesta debe mostrar evidencia: fecha comprometida, avance,
horas consumidas, presupuesto y responsable.

## Margen por Proyecto

El CEO pregunta que proyectos dejaron mejor o peor margen. El sistema debe
consultar views analiticas y evitar exponer detalles sensibles innecesarios.

## Soporte y SLA

El CEO quiere conocer tickets abiertos, tickets criticos, cumplimiento de SLA y
clientes con mayor volumen de soporte. La respuesta debe permitir tabla y grafica
por prioridad o cliente.

## Finanzas Operativas

El CEO pregunta por burn rate, runway, costos por area, margen bruto o variacion
de gastos. Si la pregunta es ambigua, el sistema debe pedir aclaracion o sugerir
metricas disponibles.

## Preguntas Sugeridas para CEO

El sistema debe proponer preguntas utiles sin depender de prompt engineering:

- "Mostrar MRR y crecimiento de los ultimos 6 meses."
- "Listar proyectos con margen menor al 20%."
- "Identificar clientes con tickets criticos abiertos."
- "Comparar pipeline actual contra el mes pasado."
- "Generar forecast de ventas del trimestre."
