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

## Composer

El composer debe mantenerse simple. Sus opciones no son filtros ni formularios
analiticos; funcionan como modos que complementan el prompt y ayudan al sistema
a entender el tipo de salida esperada. El CEO debe poder escribir una pregunta
natural sin tomar muchas decisiones antes de enviarla.

### Modos del Composer

| Modo | Uso principal | Comportamiento esperado |
| --- | --- | --- |
| `responder` | Respuesta ejecutiva directa | Modo default. Responde con texto claro y puede incluir tabla, KPI o grafica si el prompt lo pide o si aporta evidencia critica. |
| `analizar` | Explicar causas y evidencia | Prioriza analisis, comparaciones, posibles causas, datos relevantes y siguientes preguntas. Puede incluir texto, tabla y grafica. |
| `reporte_visual` | Generar salida visual | Prioriza una grafica o reporte visual con narrativa breve, `chart_spec`, tabla base, freshness y warnings. |
| `plan` | Convertir hallazgos en accion | Prioriza acciones, riesgos, prioridades y siguientes pasos. Si el prompt pide grafica, el plan debe incluir tambien una grafica relevante. |

### Precedencia entre Modo y Prompt

El `intent_mode` define la prioridad base de la respuesta, pero no limita lo que
el usuario puede pedir en el texto. El prompt puede agregar requisitos
explicitos como "analiza", "hazme una grafica", "crea un reporte visual" o
"incluye un plan".

Reglas:

- El modo establece el objetivo principal.
- Los requisitos explicitos del prompt se deben cumplir si son seguros y hay
  datos suficientes.
- Si modo y prompt piden cosas distintas, la respuesta debe combinar ambos.
- Si la combinacion requiere datos nuevos, la consulta vuelve a pasar por SQL
  Safety Layer.
- Si faltan periodo, metrica o alcance, el sistema debe pedir aclaracion.

Ejemplos:

- En `responder`, si el usuario pide "analiza la caida de MRR", se entrega
  analisis, no solo una respuesta breve.
- En `analizar`, si el usuario pide "hazme un reporte visual", se entrega
  analisis y reporte visual.
- En `plan`, si el usuario pide "incluye una grafica", el plan debe incluir una
  grafica como artefacto.
- En cualquier modo, si el usuario pide "grafica esto", se genera grafica cuando
  existan datos suficientes y seguros.

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

## Edicion Interna de Graficas

Cada grafica generada debe permitir la accion `Editar grafica`. Esta accion abre
un panel contextual con un mini chat ligado al `artifact_id` de la grafica. El
mini chat edita la configuracion visual, no ejecuta SQL por si mismo.

Cambios permitidos:

- tipo de grafica;
- titulo;
- ejes;
- series visibles;
- orden;
- agregacion visual;
- formato numerico;
- etiquetas;
- leyenda;
- colores;
- anotaciones.

Si el usuario pide cambiar datos, periodo, metrica, fuente o filtro de negocio,
el sistema debe crear una nueva solicitud en el chat principal. Esa nueva
solicitud debe pasar nuevamente por SQL Safety Layer antes de consultar datos.

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
- Recuperar definiciones desde la capa semantica.
- Mantener continuidad entre preguntas del mismo hilo.

## Seguridad y Gobernanza

- El frontend no genera ni ejecuta SQL.
- En el camino principal, el LLM produce `MetricQuery` y la capa semantica compila SQL
  determinista.
- El LLM solo produce SQL candidato en fallback Text-to-SQL, usando
  `BusinessSchemaContext` allowlisted.
- Todo SQL generado o candidato pasa por SQL Safety Layer.
- El runtime usa PostgreSQL read-only.
- El rol `CEO` limita tools, views, columnas y acciones.
- JWT se valida en cada request web.
- MCP mantiene autenticacion separada por bearer token/API key.
- Auditoria registra prompts, SQL candidato, SQL validado, rol, cliente,
  warnings y `trace_id`.
- Cada uso de fallback SQL emite log `warn` `analytics.fallback_sql_triggered` para
  evaluar si la pregunta debe convertirse en metrica gobernada.

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
