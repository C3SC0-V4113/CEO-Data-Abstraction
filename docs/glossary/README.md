# Glosario

## Agente

Sistema basado en IA que puede razonar sobre una tarea, usar herramientas,
consultar datos y producir una respuesta o accion.

## Alerta Inteligente

Notificacion generada cuando una metrica cruza un umbral, cambia de forma
anomala o requiere atencion segun reglas y contexto.

## API Gateway

Capa de borde que controla el trafico antes de llegar a un servicio: TLS, rate
limiting, throttling, cuotas, WAF/IP rules, limite de tamano y routing, mas auth
coarse y observabilidad. En este proyecto se usan **dos gateways Cloudflare**, uno
frente al backend web (Web API Gateway) y otro frente al servicio MCP (MCP API Gateway).
Ver ADR-0007.

## Auth Service-to-Service

Autenticacion entre servicios internos (no de usuario final). El servicio MCP llama a la
Core Internal API del backend con `CORE_SERVICE_TOKEN` y/o red privada; esa API no se
expone por el Web API Gateway.

## Artefacto de Chat

Resultado estructurado generado dentro de una conversacion. Puede ser una tabla,
grafica, KPI, resumen o reporte bajo demanda. Debe conservar contexto, datos,
warnings, freshness y `trace_id`.

## BusinessSchemaContext

Contexto reducido y versionado que se entrega al LLM solo en el fallback Text-to-SQL.
Incluye views `ceo_*`, columnas, grano, relaciones, filtros, funciones y reglas de
seguridad allowlisted. No es DDL crudo ni acceso a la base completa.

## Capa Semantica

Definicion gobernada de metricas, dimensiones, filtros y relaciones. Evita que
cada usuario o modelo interprete los datos de forma distinta. En este proyecto se
implementa como una **capa de metricas headless propia** (ver ADR-0006 y
`docs/architecture/semantic-layer-and-model-strategy.md`): el LLM produce un
`MetricQuery` y un generador determinista compila el SQL sobre las views `ceo_*`.

## Catalogo de Metricas

Configuracion gobernada (YAML/JSON) que define cada metrica del MVP con `name`,
`label`, `description`, `synonyms`, `grain`, `source_view`, `dimensions`,
`filters_allowed`, `format` y `default_chart`. Funciona como documentacion, allowlist
y contrato. Se entrega al LLM de forma compacta y cacheable, no como DDL crudo.

## Chart Spec

Objeto estructurado que describe como renderizar una grafica: tipo, ejes,
series, formato, etiquetas, leyenda, colores y anotaciones. Puede editarse sin
consultar datos nuevos si solo cambia la configuracion visual.

## Core Internal API

API interna del backend Fastify (`/internal/core/*`) que publica el pipeline
(Orchestrator + Capa Semantica + SQL Safety Layer + read-only + auditoria) a llamadores
internos de confianza, como el servicio MCP. Usa auth service-to-service y no se enruta
por el Web API Gateway.

## Chatbot Ejecutivo

Interfaz principal del MVP despues del login. Permite que el CEO pregunte en
lenguaje natural y reciba respuestas ejecutivas con evidencia, tablas, graficas
o KPIs generados bajo demanda.

## Copiloto

Experiencia asistida por IA que acompana al usuario dentro de una interfaz,
normalmente combinando acciones guiadas, explicaciones y conversacion.

## Fallback Text-to-SQL

Camino secundario para preguntas exploratorias fuera de la cobertura del catalogo de
metricas. El LLM genera SQL candidato (como en el enfoque original) que igual pasa por
el SQL Safety Layer y el rol read-only. Se marca y audita como `path = fallback_sql`.
Usa `BusinessSchemaContext`, no schema crudo. Cada uso debe emitir el log estructurado
`analytics.fallback_sql_triggered` para avisar que la Metric Layer no cubrio la pregunta.

## Fallback SQL Log

Evento estructurado de nivel `warn` emitido cuando el backend usa fallback SQL. El evento
se llama `analytics.fallback_sql_triggered` e incluye campos como `trace_id`,
`conversation_id`, `user_role`, `fallback_reason`, `missing_metric_or_dimension`,
`schema_context_version` y hashes de SQL. Sirve para que desarrollo y discovery decidan
si conviene ampliar el catalogo de metricas.

## Forecast

Pronostico de una metrica futura, por ejemplo ventas esperadas del proximo mes.
Debe indicar horizonte, supuestos y nivel de confianza.

## Fastify

Framework backend para Node.js usado en esta propuesta para exponer APIs,
endpoint MCP, autenticacion y el pipeline Text-to-SQL en TypeScript.

## Job

Proceso automatizado que corre bajo una frecuencia o condicion. En este repo los
jobs son una hipotesis para reportes, snapshots, alertas y analisis pesados.

## JWT

JSON Web Token usado para representar una sesion autenticada. En este MVP debe
incluir al menos usuario, expiracion y rol `CEO`, y debe validarse en backend en
cada request protegida.

## MCP

Model Context Protocol. Protocolo para exponer herramientas y contexto a modelos
o agentes como Codex y Claude. En este proyecto se sirve desde un **servicio MCP
independiente** (ya no embebido en Fastify; ver ADR-0007).

## Servicio MCP (standalone)

Servicio desplegable propio (Railway, detras del MCP API Gateway) que implementa el
protocolo MCP como adapter delgado: valida `MCP_API_KEY` y el scope por tool, y mapea cada
tool a la Core Internal API del backend. No accede a la base de datos ni porta credenciales
de DB/LLM; toda la logica de datos vive en el backend core.

## LLM Orchestrator

Modulo interno del backend que coordina el flujo Text-to-SQL: prepara contexto,
llama al modelo, recibe SQL candidato, invoca validacion, ejecuta consultas
read-only y construye la respuesta final. No es Codex ni Claude; esos son
clientes posibles via MCP.

## Metrica

Valor cuantificable de negocio, como ventas totales, margen, cumplimiento de
meta o ticket promedio.

## MetricQuery

Objeto estructurado que el LLM produce en el camino semantico: `metric`,
`dimensions`, `filters`, `time_range`, `compare_to` y `limit`. Se valida contra el
catalogo de metricas (allowlist natural) y un generador determinista lo compila a SQL.
El LLM no escribe SQL en este camino.

## MetricCatalogContext

Contexto compacto y cacheable entregado al LLM para el camino semantico. Contiene las
metricas permitidas para el rol, sinonimos, dimensiones, filtros, grano, formatos,
`source_view` y reglas del contrato `MetricQuery`. No incluye DDL crudo.

## Mini Chat de Grafica

Panel contextual ligado a un `artifact_id` de tipo `chart`. Permite pedir
cambios visuales sobre una grafica existente. Si el usuario pide cambiar datos,
periodo, metrica o fuente, debe volver al chat principal y pasar por SQL Safety
Layer.

## Modo de Composer

Opcion simple del composer que complementa el prompt y define la prioridad base
de la respuesta. No funciona como filtro. Los modos del MVP son `responder`,
`analizar`, `reporte_visual` y `plan`.

## Pregunta Sugerida

Pregunta que el sistema propone al usuario segun su contexto, rol, datos
recientes o anomalias detectadas.

## Prompt Caching

Mecanismo de los proveedores de LLM que abarata los tokens de entrada estables entre
interacciones (en este proyecto, el catalogo de metricas y el system prompt). En la
fraccion de hits de cache, esos tokens se cobran a una tarifa menor; es la principal
palanca de costo de la calculadora (`docs/cost/`).

## Prisma ORM

ORM TypeScript usado para modelos tipados, migraciones y conexion a PostgreSQL.
En runtime debe usar una credencial read-only; las consultas SQL generadas por
IA solo pueden ejecutarse despues de pasar por el SQL Safety Layer.

## RAG

Retrieval-Augmented Generation. Patron donde el modelo recupera informacion
relevante desde documentos o bases de conocimiento antes de responder. En este MVP queda
fuera del alcance del pipeline de Metric Layer; solo se evaluaria como evolucion futura
si el catalogo crece mas alla de lo razonable para prompt caching.

## Requisito Explicito del Prompt

Solicitud escrita directamente por el usuario, por ejemplo "analiza", "haz una
grafica", "crea un reporte visual" o "incluye un plan". Debe combinarse con el
modo de composer si es seguro y hay datos suficientes.

## Rol CEO

Rol inicial y unico del MVP. Aunque solo exista un usuario operativo, el backend
debe validar el rol para limitar endpoints, tools, views, columnas y acciones.

## Routing de Modelos (dos niveles)

Estrategia para optimizar costo/calidad usando dos perfiles de modelo: un **nivel
ligero** barato para clasificacion de intencion, aclaraciones, `chart_spec` y
sugerencias; y un **nivel planificador** fuerte para NL -> `MetricQuery`, narrativa y
analisis. El modelo se elige por configuracion y es intercambiable entre proveedores.

## Snapshot

Resultado precomputado de una metrica en un momento o periodo. En el paradigma
chatbot-first ya **no es una superficie separada** (no hay dashboards ni historico de
reportes en el MVP): si se introduce, sera como cache interna o como job futuro
(`docs/strategy/automated-insights.md`), no como vista propia.

## SQL Libre

SQL generado dinamicamente por un LLM sin restricciones suficientes. Se considera
riesgoso para este caso por seguridad, consistencia y auditabilidad.

## SQL Safety Layer

Capa que valida SQL antes de ejecutarlo. Debe permitir solo `SELECT`, usar parser
AST, bloquear multiples statements, aplicar allowlist de views/tablas/columnas y
forzar limites de filas y tiempo.
