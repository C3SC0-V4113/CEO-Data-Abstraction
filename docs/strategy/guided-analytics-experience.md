# Guided Analytics Experience

## Idea Central

La experiencia principal deja de ser report-first. La aplicacion web debe ser
chat-first: despues del login, el CEO entra a una unica vista de chatbot donde
pregunta, recibe explicaciones ejecutivas y ve reportes generados bajo demanda
dentro de la conversacion.

El sistema sigue siendo guiado, pero la guia ocurre dentro del chat mediante
preguntas sugeridas, acciones rapidas, aclaraciones, filtros conversacionales y
artefactos visuales generados por cada respuesta. No hay dashboard principal ni
pantallas separadas de reportes para el MVP.

## Superficie de Interfaz

### Login

La primera pantalla es un login obligatorio:

- Usuario CEO creado por seed/setup.
- Autenticacion con JWT.
- Sin registro publico.
- Sin autoservicio de usuarios.
- Rol `CEO` incluido en claims y validado en backend.

### Chatbot Ejecutivo

La segunda y principal pantalla es el chatbot:

- Composer para preguntas en lenguaje natural.
- Preguntas sugeridas visibles antes y despues de cada respuesta.
- Respuestas ejecutivas con tablas, graficas o KPIs embebidos.
- Reportes generados bajo demanda dentro del hilo.
- Filtros conversacionales como periodo, cliente, proyecto, area o metrica.
- Historial minimo de conversacion para mantener contexto.
- Estado de freshness y advertencias de calidad de datos por respuesta.
- `trace_id` visible o recuperable para auditoria.

## Reportes Dentro del Chat

Un reporte ya no es una vista SSR separada. Es un artefacto generado por una
respuesta del chatbot.

Ejemplos:

- El CEO pregunta "Mostrar MRR y crecimiento de los ultimos 6 meses".
- El sistema responde con resumen ejecutivo, grafica de linea y tabla resumida.
- El CEO pregunta "Que explica la caida de marzo?"
- El sistema conserva contexto del reporte anterior y profundiza.

Cada artefacto de reporte debe incluir:

- `artifact_id`
- `conversation_id`
- `question`
- `period`
- `source_views`
- `validated_sql`
- `summary`
- `data`
- `chart_spec`
- `freshness`
- `warnings`
- `trace_id`

## Patrones de Guia

### Preguntas Sugeridas

El sistema propone preguntas utiles para reducir prompt engineering:

- "Mostrar MRR y crecimiento de los ultimos 6 meses."
- "Listar proyectos con margen menor al 20%."
- "Identificar clientes con tickets criticos abiertos."
- "Comparar pipeline actual contra el mes pasado."
- "Generar forecast de ventas del trimestre."
- "Explicar cambios principales del periodo."
- "Revisar calidad de datos antes de responder."

### Acciones Rapidas

Las acciones rapidas existen dentro del chat, asociadas al contexto actual:

- Comparar.
- Explicar.
- Forecast.
- Ver detalle.
- Cambiar periodo.
- Cambiar metrica.
- Descargar resultado, si se aprueba despues.

### Aclaraciones

Si una pregunta es ambigua, el chatbot debe pedir aclaracion antes de generar SQL
innecesario. Ejemplos:

- Periodo no especificado.
- Metrica ambigua.
- Cliente o proyecto con nombres similares.
- Solicitud que podria exponer datos fuera del rol.

### Conversacion con Estado

El estado conversacional debe conservar:

- Periodo seleccionado.
- Cliente, proyecto o area filtrada.
- Metrica activa.
- Ultimo artefacto generado.
- Comparacion previa.
- Rol y permisos.
- Advertencias de calidad de datos.

## Rol de IA

- Traducir intencion ejecutiva a consultas seguras.
- Pedir aclaraciones cuando falte contexto.
- Generar explicaciones en lenguaje natural.
- Proponer preguntas siguientes.
- Generar especificaciones de visualizacion.
- Detectar anomalias con evidencia.
- Recuperar definiciones desde capa semantica o RAG.
- Mantener continuidad entre preguntas del mismo hilo.

## Seguridad y Gobernanza

- El frontend no genera ni ejecuta SQL.
- El LLM solo produce SQL candidato.
- Todo SQL candidato pasa por SQL Safety Layer.
- El runtime usa PostgreSQL read-only.
- El rol `CEO` limita tools, views, columnas y acciones.
- JWT se valida en cada request web.
- MCP mantiene autenticacion separada por bearer token/API key.
- Auditoria registra prompts, SQL candidato, SQL validado, rol, cliente,
  warnings y `trace_id`.

## Como Reduce Prompt Engineering

- El chat sugiere preguntas relevantes desde el inicio.
- Las acciones rapidas convierten operaciones comunes en comandos guiados.
- La capa semantica evita que el CEO recuerde nombres de tablas o formulas.
- El sistema pide aclaraciones en vez de inventar supuestos criticos.
- Los reportes se generan en el flujo conversacional, no en pantallas separadas.

## Riesgos

- Un chat vacio puede seguir intimidando si no hay buenas sugerencias iniciales.
- Demasiada libertad puede producir preguntas ambiguas o imposibles.
- Si el estado conversacional se maneja mal, las respuestas pueden mezclar
  periodos o filtros.
- Los artefactos generados pueden parecer reportes formales aunque dependan de
  datos ficticios o freshness limitada.

## Recomendacion Inicial

Disenar primero una aplicacion de dos pantallas: login y chatbot ejecutivo. El
chatbot debe cubrir preguntas sugeridas, aclaraciones, respuestas con tablas y
graficas embebidas, y auditoria completa. Los dashboards, reportes persistentes
y envios automaticos quedan como extensiones futuras sujetas a ADR.
