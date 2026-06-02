# Problem Framing

## Requerimiento Actualizado

El requerimiento pide un sistema de consulta de datos asistido por IA
Text-to-SQL, accesible desde dos frentes:

- Una aplicacion web propietaria con login y chatbot, capaz de renderizar tablas
  y graficas dinamicas dentro de la conversacion.
- Un servidor MCP remoto consumible por Claude Desktop, Cursor/Codex u otros
  clientes compatibles.

El sistema debe permitir consultas complejas en lenguaje natural y mantener una
postura estricta de seguridad: autenticacion robusta, conexion de base de datos
read-only y mitigacion explicita contra SQL injection.

## Usuario Objetivo

El usuario objetivo sera el CEO de una empresa desarrolladora de software. Este
usuario no quiere aprender SQL ni conocer la estructura de tablas. Quiere
entender la salud de la compania con preguntas ejecutivas sobre revenue,
clientes, proyectos, delivery, soporte, finanzas operativas y forecast.

## Problema Real

El problema no es solamente "hablar con la data". El problema es dar una
interfaz confiable para que una persona ejecutiva convierta preguntas ambiguas
en respuestas accionables, visualizaciones y evidencia verificable.

Ejemplos:

- "Como va el MRR este mes?"
- "Que proyectos estan atrasados o en riesgo?"
- "Cual es nuestro churn?"
- "Cuanto runway tenemos?"
- "Que clientes tienen mas tickets criticos?"
- "Cual es el forecast de ventas del trimestre?"

## Objetivo de la Aplicacion

Construir una base de arquitectura para un producto que:

- Traduzca lenguaje natural a SQL de solo lectura.
- Use Next.js SSR + shadcn/ui para una experiencia web ejecutiva basada en
  login y chatbot.
- Exponga las mismas capacidades por MCP remoto.
- Genere tablas y graficas dinamicas desde los resultados.
- Audite cada consulta y cada SQL generado.
- Proteja la base con usuario read-only, allowlists y validacion SQL.

## Principio de Diseno

El LLM puede generar SQL candidato, pero nunca debe ejecutarse sin validacion.
La seguridad no depende solo del prompt: debe existir un SQL Safety Layer y un
rol de base de datos con permisos estrictos de lectura.
