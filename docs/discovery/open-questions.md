# Open Questions

## Negocio

- Para el MVP se asume CEO como usuario principal. Queda abierto si despues se
  incluyen CFO, COO y lideres de area.
- Que metricas ejecutivas son oficiales: MRR, ARR, churn, margen, runway,
  pipeline, SLA?
- Que metricas son oficiales y cuales son exploratorias?
- Que decisiones espera tomar el usuario despues de recibir una respuesta?

## Datos

- Para el MVP se asume PostgreSQL con datos ficticios generados por seed. Queda
  abierto si despues se conecta a base transaccional, data warehouse o archivos.
- Para el MVP se generaran 12 a 18 meses de historico ficticio. Queda abierto si
  historico real sera suficiente para forecast.
- Existen definiciones formales para MRR, ARR, churn, expansion revenue y margen
  por proyecto?
- Como se manejan monedas, descuentos, credit notes y contratos cancelados?
- Que tan confiable es la calidad de datos?

## IA y Seguridad

- Que herramientas MCP puede ejecutar cada rol?
- Que datos no deben exponerse al agente?
- Como se auditan preguntas, herramientas llamadas y respuestas?
- Se requiere aprobacion humana para reportes enviados automaticamente?

## Experiencia

- Para el MVP se asume dashboard/reportes como experiencia principal y chat como
  soporte secundario.
- Que consultas deberian ser sugeridas por defecto?
- Que automatizaciones realmente ahorran tiempo y cuales podrian generar ruido?
- Como se medira si se redujo la dependencia del chat?
