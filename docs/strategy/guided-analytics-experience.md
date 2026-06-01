# Guided Analytics Experience

## Idea Central

Si el usuario no sabe que preguntar, la solucion debe guiarlo. El chat existe,
pero queda en segundo plano. La experiencia principal sera un dashboard
ejecutivo con reportes y graficas predefinidas.

## Patrones de Interfaz

### Catalogo de Metricas

Mostrar metricas disponibles con nombres entendibles:

- MRR del mes.
- ARR.
- Churn.
- Pipeline comercial.
- Proyectos en riesgo.
- Margen por proyecto.
- Tickets criticos.
- Runway.
- Forecast.

Cada metrica deberia incluir definicion, formula, filtros permitidos y fecha de
actualizacion.

### Preguntas Sugeridas

El sistema propone preguntas segun contexto:

- "Comparar ventas contra el mes anterior."
- "Ver MRR y crecimiento de los ultimos 6 meses."
- "Listar proyectos con margen menor al 20%."
- "Mostrar clientes con tickets criticos abiertos."
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

### Dashboard Report-First

La app Next.js SSR debe separar graficas por vistas: overview, revenue,
customers, pipeline, projects, support, finance e historico de reportes. El CEO
primero ve reportes; luego puede abrir el chat si necesita preguntar algo.

### Chat Contextual por Reporte

Cada grafico o reporte debe tener un boton `Preguntar`. Al usarlo, se abre el
sidebar o burbuja de chat y se inyecta contexto del reporte:

- `report_id`
- filtros activos;
- metricas visibles;
- tipo de grafico;
- resumen del reporte;
- muestra de datos;
- view origen;
- freshness.

### Conversacion con Estado

Cuando el usuario si use chat, el sistema debe conservar contexto:

- Periodo seleccionado.
- Cliente, proyecto o equipo filtrado.
- Metrica activa.
- Reporte o grafico activo.
- Comparacion previa.
- Rol y permisos.

## Como Reduce Prompt Engineering

- El usuario selecciona intenciones en vez de escribir prompts largos.
- Las preguntas sugeridas convierten datos recientes en acciones posibles.
- La capa semantica evita que el usuario recuerde nombres de tablas o formulas.
- El copiloto puede pedir aclaraciones cuando la pregunta es ambigua.
- Los reportes predefinidos resuelven las preguntas comunes sin abrir el chat.

## Rol de IA

- Generar explicaciones en lenguaje natural.
- Sugerir preguntas siguientes.
- Detectar anomalias.
- Resumir reportes.
- Traducir intencion a herramientas MCP cuando se use un cliente externo.
- Recuperar definiciones desde RAG.

## Riesgos

- Una interfaz demasiado guiada puede limitar analisis exploratorio.
- Demasiadas sugerencias pueden distraer.
- Si el sistema sugiere preguntas irrelevantes, pierde confianza.

## Recomendacion Inicial

Disenar primero un dashboard pequeno con reportes ejecutivos y boton `Preguntar`
por grafico. Luego validar si el CEO necesita mas chat libre o si los reportes
cubren la mayor parte de los casos.
