# ADR-0006: Adoptar Capa Semantica de Metricas Headless y Estrategia de Modelos

## Status

Proposed

## Date

2026-06-02

## Context

ADR-0005 fijo la experiencia chatbot-first: el CEO pregunta en lenguaje natural y
recibe narrativa, KPIs, tablas y graficas como artefactos dentro del chat. El
pipeline documentado en `docs/architecture/proposal.md` asume Text-to-SQL: el LLM
genera SQL candidato y el SQL Safety Layer lo valida antes de ejecutarlo con un rol
read-only.

El problema es la **exactitud y la confianza**. El CEO usara estas cifras para
decidir (priorizar proyectos, revisar churn, evaluar runway). Text-to-SQL libre
falla de forma silenciosa: produce un numero plausible pero incorrecto (join mal
hecho, metrica mal interpretada, filtro omitido) que nadie nota hasta que la
decision ya se tomo. La evidencia publica compartida lo confirma:

- Benchmark dbt LLM Semantic Layer (2026): con datos modelados, Claude Sonnet 4.6
  pasa de 90.0% (Text-to-SQL) a 98.2% via capa semantica; GPT-5.3 Codex pasa de
  84.1% a 100%. Mas importante que el numero: fuera de cobertura la capa semantica
  **devuelve error** en vez de un resultado enganoso.
- Denodo: con metadata semantica sobre el schema, BIRD sube de ~20% a ~87% y Spider
  de ~50% a ~83%. La lectura util para este MVP es que la metadata de negocio importa
  mas que exponer tablas crudas al modelo.
- BIRD-bench: expertos humanos ~92.96%; los mejores sistemas ~82%. El reto no es
  "hablar con los datos" sino incorporar logica de negocio que no esta en el schema.

Ademas, faltaba definir tres cosas que el negocio pidio explicitamente: que
modelo(s) usar y por que, como entregar el schema al LLM, y cuanto costara.

El stack vigente (ADR-0002) es Fastify + Prisma + PostgreSQL en Railway, sin data
warehouse. No hay infraestructura dbt/MetricFlow ni se justifica para un MVP de un
solo CEO.

## Decision Drivers

- Exactitud confiable de las cifras ejecutivas (fallar en voz alta, no en silencio).
- Mantener la gobernanza ya decidida: SQL Safety Layer, allowlists, rol read-only,
  auditoria.
- Encajar en el stack actual sin agregar infraestructura pesada (no dbt/warehouse).
- Reducir y controlar el costo de tokens del LLM.
- Mantener un solo contrato compartido entre la web y el canal MCP.
- No acoplar la arquitectura a un unico proveedor de modelos.

## Considered Options

### Option 1: Text-to-SQL libre validado por SQL Safety Layer

- Pros: maxima flexibilidad; cubre preguntas exploratorias arbitrarias; sin catalogo
  que mantener.
- Cons: falla en silencio con cifras plausibles pero erroneas; exige enviar mucho
  schema al LLM (costo de tokens alto); exactitud ~84-90% en el mejor caso.

### Option 2: dbt Semantic Layer (MetricFlow)

- Pros: estandar de industria; SQL determinista; alineado con el benchmark.
- Cons: agrega dbt + un estilo warehouse y peso operativo desproporcionado para un
  MVP de un CEO sobre PostgreSQL en Railway.

### Option 3: Capa de metricas headless propia + fallback Text-to-SQL

- Pros: el LLM elige metrica/dimension/filtro (tarea mas facil y barata que escribir
  SQL), un generador determinista compila el SQL sobre las views `ceo_*`; el catalogo
  es la allowlist natural; falla en voz alta; el catalogo compacto se cachea y abarata
  tokens; mantiene un contrato unico web/MCP; el fallback cubre lo exploratorio.
- Cons: hay que disenar y mantener el catalogo de metricas y el generador; las
  preguntas fuera de cobertura caen al fallback (aun gobernado).

## Decision

Adoptar la **Option 3**: una capa semantica de **metricas headless propia** entre el
LLM Orchestrator y el SQL Safety Layer.

- El flujo por defecto es **lenguaje natural -> `MetricQuery` -> SQL determinista**,
  no lenguaje natural -> SQL libre. El LLM produce un `MetricQuery` (metrica,
  dimensiones, filtros, rango temporal, comparacion, limite) validado contra el
  catalogo de metricas; un generador determinista lo compila a SQL sobre las views
  `ceo_*`.
- El catalogo de metricas (YAML/JSON gobernado) define metricas, dimensiones, filtros
  permitidos, grano, sinonimos, `source_view`, formato y grafico por defecto. La
  validacion contra el catalogo funciona como allowlist natural.
- **Text-to-SQL libre queda como fallback explicito y auditado** solo para preguntas
  exploratorias fuera de cobertura del catalogo. El fallback sigue pasando por el SQL
  Safety Layer y el rol read-only; se marca en la auditoria como `path = fallback_sql`.
- **Estrategia de entrega de schema:** no se vuelca el DDL completo al LLM. El camino
  semantico entrega un `MetricCatalogContext` compacto y cacheable (prompt caching). El
  fallback Text-to-SQL entrega un `BusinessSchemaContext` reducido: solo views `ceo_*`,
  columnas, relaciones, filtros y funciones allowlisted, con descripciones de negocio y
  ejemplos cortos. El detalle on-demand via `describe_business_schema` /
  `GET /api/schema/catalog` devuelve catalogo permitido por rol, no schema crudo.
- Cada caida a fallback SQL debe emitir un log estructurado `warn` con evento
  `analytics.fallback_sql_triggered`. Este evento avisa al equipo de desarrollo que la
  Metric Layer no cubrio la pregunta y alimenta la decision de ampliar el catalogo de
  metricas.
- **Estrategia de modelos:** arquitectura agnostica de proveedor con **routing de dos
  niveles**: un modelo ligero/barato para clasificacion de intencion, aclaraciones,
  `chart_spec` y sugerencias; un modelo planificador fuerte para NL->`MetricQuery`,
  narrativa y analisis. El modelo se elige por configuracion (`ORCHESTRATOR_MODEL`,
  `LIGHT_MODEL`) y es intercambiable. **Modelos definidos por default: GPT-5.2
  (planificador) + GPT-5 mini (ligero)**, ambos OpenAI, decididos cruzando exactitud
  (la familia GPT-5.x lidera el benchmark dbt de capa semantica) y costo (la calculadora
  da ~$331/mes a 100 usuarios para este par, ~22% menos que Claude Sonnet/Haiku).
  Gemini 2.5 Pro/Flash es la alternativa mas barata (~$259/mes) pendiente de una eval de
  exactitud; Claude Sonnet 4.6/Haiku 4.5 la de mayor exactitud publicada no-Codex. Detalle
  y fuentes en `docs/architecture/semantic-layer-and-model-strategy.md` y
  `docs/cost/README.md`.

El detalle de diseno (esquema del catalogo, contrato `MetricQuery`, generador SQL,
entrega de schema, fallback gobernado, logs y comparativa de modelos) vive en
`docs/architecture/semantic-layer-and-model-strategy.md`.

## Rationale

Una capa de metricas headless captura casi toda la ganancia de exactitud del
benchmark sin el peso de dbt/MetricFlow ni un warehouse. Reduce la tarea del LLM de
"escribir SQL correcto" a "elegir la metrica y dimension correctas", que es mas facil,
mas barata en tokens y permite usar modelos mas economicos sin perder exactitud. El
modo de fallo cambia de "numero incorrecto silencioso" a "error o aclaracion", que es
lo correcto para datos que sustentan decisiones de un CEO. El fallback Text-to-SQL
conserva flexibilidad para lo exploratorio sin renunciar a la gobernanza. Headless
(API-first) permite que la web y el canal MCP compartan el mismo contrato. Mantener la
capa de modelos por configuracion evita el lock-in y habilita una comparacion de costo
honesta entre proveedores.

## Consequences

### Positive

- Cifras ejecutivas mas exactas y con modo de fallo seguro.
- Menos tokens de schema enviados al LLM (catalogo compacto y cacheable) -> menor costo.
- El catalogo de metricas es a la vez documentacion, allowlist y contrato.
- Mismo pipeline y contrato para web y MCP.
- Libertad para elegir/cambiar de modelo y proveedor segun costo y calidad.

### Negative

- Hay que disenar y mantener el catalogo de metricas y el generador SQL determinista.
- Preguntas fuera de cobertura dependen del fallback Text-to-SQL (menor exactitud).
- Dos caminos (semantico y fallback) aumentan la superficie a probar y auditar.

### Risks and Mitigations

- Riesgo: cobertura insuficiente del catalogo frustra al CEO.
  Mitigacion: arrancar con las metricas oficiales del MVP; medir cuantas preguntas
  caen al fallback y promover las recurrentes a metricas del catalogo.
- Riesgo: el LLM produce un `MetricQuery` invalido o ambiguo.
  Mitigacion: validar contra el catalogo, pedir aclaracion antes de ejecutar y
  registrar el intento en auditoria.
- Riesgo: el fallback reintroduce SQL libre riesgoso.
  Mitigacion: sigue pasando por SQL Safety Layer, allowlists y rol read-only; se marca
  y audita como camino de fallback. Ademas emite un log estructurado para que desarrollo
  revise si la pregunta debe convertirse en una metrica oficial.
- Riesgo: la estimacion de costo es incierta (precios cambian).
  Mitigacion: calculadora `.xlsx` con precios editables y supuestos de tokens
  explicitos; revisar precios reales antes de comprometer presupuesto.

## Implementation Notes

- El catalogo de metricas se versiona como configuracion (YAML/JSON) y se carga al
  iniciar el backend; cambios pasan por revision.
- El `MetricQuery` se valida contra el catalogo antes de compilar SQL; el SQL generado
  sigue pasando por el SQL Safety Layer (AST, allowlist, `LIMIT`, timeout, read-only).
- La auditoria registra el `path` (`semantic` o `fallback_sql`), el `MetricQuery`,
  `fallback_reason`, `fallback_schema_context_version`, el SQL candidato, el SQL
  validado y el `trace_id`.
- El backend emite un log `warn` `analytics.fallback_sql_triggered` cuando usa
  `fallback_sql`. Campos minimos: `trace_id`, `conversation_id`, `user_role`,
  `question`, `fallback_reason`, `missing_metric_or_dimension`,
  `schema_context_version`, `candidate_sql_hash`, `validated_sql_hash` y `timestamp`.
  No debe incluir SQL completo si contiene valores sensibles; usar hashes o una version
  sanitizada.
- Variables de configuracion sugeridas: `ORCHESTRATOR_MODEL` (default `GPT-5.2`),
  `LIGHT_MODEL` (default `GPT-5 mini`), `LLM_PROVIDER` (default `openai`), ademas de las
  claves por proveedor. Cambiar de modelo/proveedor no debe requerir cambios de codigo
  fuera de la capa de configuracion.
- La tool MCP `describe_business_schema` debe devolver el catalogo de metricas
  permitido para el rol, no el DDL crudo.

## Related Decisions

- ADR-0002: Next.js SSR, Fastify, Prisma y MCP remoto para Text-to-SQL. Este ADR
  refina su pipeline (no lo supersede): agrega la capa semantica antes del SQL Safety
  Layer.
- ADR-0005: Chatbot-first guided analytics. Este ADR provee la capa semantica que
  sostiene la exactitud de los artefactos generados en el chat.

## References

- `docs/architecture/semantic-layer-and-model-strategy.md`
- `docs/architecture/proposal.md`
- `docs/cost/llm-cost-calculator.xlsx`
- dbt LLM Semantic Layer Benchmark: https://dbt-labs.github.io/dbt-llm-sl-bench/
- dbt: Semantic Layer vs Text-to-SQL (2026):
  https://docs.getdbt.com/blog/semantic-layer-vs-text-to-sql-2026
- Denodo: semantic layer for LLM text-to-SQL:
  https://www.datamanagementblog.com/improving-the-accuracy-of-llm-based-text-to-sql-generation-with-a-semantic-layer-in-the-denodo-platform/
- BIRD-bench overview: https://beehive-advisors.com/blog/bird-bench
- Headless vs native semantic layer:
  https://venturebeat.com/ai/headless-vs-native-semantic-layer-the-architectural-key-to-unlocking-90-text
- Evaluating LLMs for Text-to-SQL with complex workloads:
  https://arxiv.org/html/2407.19517v1
- Precios OpenAI (GPT-5.x): https://benchlm.ai/blog/posts/openai-api-pricing (2026-04-13)
- Precios Anthropic (Claude 4.x): https://benchlm.ai/blog/posts/claude-api-pricing (abr-2026)
- Precios Google Gemini 2.5: https://ai.google.dev/gemini-api/docs/pricing
