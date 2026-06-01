# Guided Analytics Experience

## Idea Central

Si el usuario no sabe que preguntar, la solucion debe guiarlo. El chat puede
seguir existiendo, pero no debe ser la unica puerta de entrada.

## Patrones de Interfaz

### Catalogo de Metricas

Mostrar metricas disponibles con nombres entendibles:

- Ventas del mes.
- Cumplimiento de meta.
- Top vendedores.
- Comparativa mensual.
- Forecast.
- Clientes con mayor crecimiento.

Cada metrica deberia incluir definicion, formula, filtros permitidos y fecha de
actualizacion.

### Preguntas Sugeridas

El sistema propone preguntas segun contexto:

- "Comparar ventas contra el mes anterior."
- "Ver vendedores bajo meta."
- "Explicar cambios principales."
- "Generar resumen ejecutivo."
- "Revisar calidad de datos."

### Acciones Rapidas

Botones o comandos predefinidos para tareas repetidas:

- Comparar.
- Explicar.
- Forecast.
- Exportar reporte.
- Programar alerta.
- Ver detalle.

### Dashboard + Copiloto

Una futura app React puede combinar graficas, tablas, filtros y chat contextual.
El usuario navega visualmente y usa IA para explicar, resumir o sugerir
siguientes pasos.

### Conversacion con Estado

Cuando el usuario si use chat, el sistema debe conservar contexto:

- Periodo seleccionado.
- Vendedor o equipo filtrado.
- Metrica activa.
- Comparacion previa.
- Rol y permisos.

## Como Reduce Prompt Engineering

- El usuario selecciona intenciones en vez de escribir prompts largos.
- Las preguntas sugeridas convierten datos recientes en acciones posibles.
- La capa semantica evita que el usuario recuerde nombres de tablas o formulas.
- El copiloto puede pedir aclaraciones cuando la pregunta es ambigua.

## Rol de IA

- Generar explicaciones en lenguaje natural.
- Sugerir preguntas siguientes.
- Detectar anomalias.
- Resumir reportes.
- Traducir intencion a herramientas MCP.
- Recuperar definiciones desde RAG.

## Riesgos

- Una interfaz demasiado guiada puede limitar analisis exploratorio.
- Demasiadas sugerencias pueden distraer.
- Si el sistema sugiere preguntas irrelevantes, pierde confianza.

## Recomendacion Inicial

Disenar primero un catalogo pequeno de metricas y preguntas sugeridas para
Ventas CRM. Luego validar si los usuarios siguen necesitando chat libre o si las
acciones guiadas cubren la mayor parte de los casos.
