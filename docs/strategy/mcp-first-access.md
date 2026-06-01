# MCP-first Access

## Proposito

MCP permite exponer herramientas de datos a agentes como Codex o Claude sin
entregar acceso directo e irrestricto a la base. La idea es que el agente use
operaciones gobernadas, auditables y alineadas con metricas de negocio.

## Principios

- No exponer SQL libre como interfaz principal.
- Definir herramientas con parametros claros.
- Limitar herramientas por rol y permiso.
- Devolver metadata de trazabilidad: metrica, periodo, filtros, fuente y fecha
  de calculo.
- Separar datos estructurados de conocimiento documental usado por RAG.

## Herramientas MCP Candidatas

### list_metrics

Devuelve metricas disponibles, descripcion, dimensiones y filtros permitidos.

### query_metric

Consulta una metrica gobernada para un periodo y conjunto de filtros.

### compare_periods

Compara una metrica entre dos periodos y devuelve diferencia absoluta,
diferencia porcentual y explicacion basica.

### rank_sellers

Ordena vendedores por una metrica definida: ventas, margen, cumplimiento de meta
o crecimiento.

### generate_forecast

Genera un pronostico sobre una metrica y horizonte permitido, indicando supuestos
y confianza.

### schedule_report

Programa un reporte recurrente si el usuario tiene permisos.

### get_report

Recupera reportes generados, resumenes y evidencia.

## Flujo Conceptual

1. Usuario o agente expresa una intencion.
2. El agente identifica si existe una herramienta o metrica adecuada.
3. Si falta informacion, el agente pide aclaracion o sugiere opciones.
4. La herramienta consulta la capa semantica y datos autorizados.
5. El agente genera respuesta con narrativa, evidencia y siguientes preguntas.

## Relacion con RAG

MCP consulta datos estructurados. RAG complementa con contexto documental:

- Definiciones de metricas.
- Politicas de ventas.
- Reglas de comision.
- Explicaciones de forecast.
- Notas de campanas o cambios comerciales.

El agente debe citar o resumir ese contexto cuando afecte la interpretacion.

## Seguridad y Gobernanza

- Autenticacion por usuario o rol.
- Allowlist de herramientas.
- Validacion de parametros.
- Limites de granularidad y volumen.
- Auditoria de herramientas llamadas.
- Registro de respuestas generadas.

## Resultado Esperado

MCP-first crea una base segura para IA sobre datos. No resuelve por si solo la
experiencia de usuario, pero permite construir chat, reportes, alertas y
dashboards sobre una capa confiable.
