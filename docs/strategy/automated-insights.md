# Automated Insights

## Hipotesis

Los jobs automaticos pueden reducir la necesidad de hablar con un chat cuando
las preguntas son recurrentes, pesadas o previsibles.

Esta no es una decision cerrada. Es una propuesta de brainstorming que debe
evaluarse contra otras opciones como preguntas sugeridas, catalogos de metricas
y dashboards guiados.

## Casos Donde Ayudan

- Reporte semanal de ventas por equipo.
- Avance quincenal contra metas.
- Cierre mensual con comparativa contra mes anterior.
- Analisis trimestral de tendencias y forecast.
- Alertas por vendedores bajo meta.
- Deteccion de productos, regiones o clientes con cambios anomalos.
- Resumen ejecutivo para direccion.

## Casos Donde Pueden Ser Excesivos

- Consultas exploratorias que cambian cada dia.
- Equipos que no revisan reportes enviados automaticamente.
- Metricas poco confiables o mal definidas.
- Alertas sin accion clara.
- Reportes que duplican dashboards existentes.

## Tipos de Jobs

### Snapshot de Metricas

Precalcula ventas, margen, cumplimiento, rankings y variaciones. Reduce costo y
latencia para consultas frecuentes.

### Reporte Narrativo con IA

Convierte metricas en una explicacion: que cambio, que lo explica y que revisar
despues.

### Alerta Inteligente

Se activa por umbral, tendencia, anomalia o cambio relevante. Debe incluir
evidencia y accion sugerida.

### Forecast Programado

Genera pronosticos bajo un horizonte definido. Debe mostrar supuestos,
confianza, historico usado y advertencias.

### Calidad de Datos

Detecta ventas sin vendedor, ordenes anuladas fuera de patron, metas faltantes o
datos inconsistentes antes de que el agente responda.

## Frecuencias a Evaluar

- Semanal: pulso operativo.
- Quincenal: avance contra metas.
- Mensual: cierre, comparativas y reporte ejecutivo.
- Trimestral: tendencias, forecast y planificacion.

## Criterios para Adoptar un Job

- La pregunta se repite con frecuencia.
- El resultado tiene una audiencia clara.
- Hay una accion esperada despues del insight.
- La metrica esta definida y validada.
- El costo de calcular bajo demanda es alto.
- El reporte reduce conversaciones innecesarias.

## Riesgos

- Saturacion de notificaciones.
- Narrativas de IA demasiado confiadas.
- Reportes que nadie lee.
- Falta de permisos por audiencia.
- Automatizaciones basadas en datos incorrectos.

## Mitigaciones

- Empezar con pocos reportes.
- Medir aperturas, uso y acciones tomadas.
- Permitir opt-in por rol o equipo.
- Incluir trazabilidad de fuentes y metricas.
- Mantener revision humana para reportes sensibles.
