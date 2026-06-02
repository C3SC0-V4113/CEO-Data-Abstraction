# Capa Semantica, Entrega de Schema y Estrategia de Modelos

Documento de diseno de detalle para ADR-0006. Cubre tres temas que el negocio pidio
cerrar: como aplicamos correctamente una capa semantica, como le damos el schema al
LLM, y que modelo(s) usamos y por que. La estimacion de costo asociada vive en
`docs/cost/` (calculadora `.xlsx`).

---

## 1. Por que una capa semantica y no Text-to-SQL libre

El paradigma chatbot-first (ADR-0005) hace que el CEO decida sobre las cifras que le
devuelve el chat. El riesgo principal de Text-to-SQL libre no es que falle: es que
**falla en silencio**, devolviendo un numero plausible pero incorrecto (un join mal
hecho, un filtro omitido, una metrica mal interpretada). Una capa semantica cambia el
modo de fallo: si la pregunta esta fuera de cobertura, responde con un error o una
aclaracion en vez de un dato enganoso.

Evidencia publica que sustenta la decision:

| Fuente | Metrica | Text-to-SQL crudo | Con capa/metadata semantica |
| --- | --- | --- | --- |
| dbt LLM SL Bench 2026 | Exactitud (Claude Sonnet 4.6) | 90.0% | 98.2% |
| dbt LLM SL Bench 2026 | Exactitud (GPT-5.3 Codex) | 84.1% | 100% |
| Denodo | BIRD | ~20% | ~87% |
| Denodo | Spider | ~50% | ~83% |
| BIRD-bench | Expertos humanos vs mejores sistemas | sistemas ~82% | humanos ~92.96% |

Lecturas clave para nuestro diseno:

- "With text-to-SQL, failure looks like a plausible but incorrect answer. With the
  Semantic Layer, failure looks like an error message." (dbt) -> elegimos fallar en
  voz alta para cifras que sustentan decisiones.
- La capa semantica tiene limites ("too many hops": joins que la capa no expresa) ->
  por eso conservamos un **fallback Text-to-SQL gobernado** para lo exploratorio.
- BIRD muestra que mucho del error viene de logica de negocio que no esta en el
  schema (p. ej. "solo cuentas OWNER califican") -> esa logica vive en el catalogo de
  metricas, no en el prompt suelto.

### Headless vs native

Una capa semantica *native* vive embebida en una herramienta BI (Looker, Power BI) y
solo sirve a esa herramienta. Una capa *headless* es API-first y desacoplada: la misma
definicion de metricas sirve a cualquier consumidor. Elegimos **headless** porque
tenemos dos consumidores del mismo contrato: la web chat-first y el canal MCP remoto.
La capa de metricas se expone como servicio interno y ambos la usan igual.

### Por que no dbt/MetricFlow (todavia)

El benchmark usa dbt MetricFlow, pero adoptarlo agrega dbt y un estilo warehouse
desproporcionado para un MVP de un CEO sobre PostgreSQL en Railway. Replicamos su idea
central (definiciones gobernadas + SQL determinista) con una capa propia ligera sobre
las views `ceo_*` que ya existen en la propuesta. Si el proyecto escala a multiples
roles y fuentes, migrar a dbt Semantic Layer queda como evolucion natural.

---

## 2. Diseno de la capa de metricas headless

Tres piezas: **catalogo de metricas** (definicion gobernada), **MetricQuery**
(lo que el LLM produce) y **generador SQL determinista** (lo que compila a SQL).

```text
Pregunta NL
   |
   v
[LLM planificador] --(MetricQuery)--> [Validacion contra catalogo]
                                            |
                                            v
                                  [Generador SQL determinista]
                                            |
                                            v
                                  [SQL Safety Layer (AST, LIMIT, timeout)]
                                            |
                                            v
                                  [PostgreSQL read-only, views ceo_*]
```

### 2.1 Catalogo de metricas

Configuracion versionada (YAML/JSON) que define cada metrica oficial del MVP. Es a la
vez documentacion, allowlist y contrato. Ejemplo:

```yaml
metrics:
  - name: mrr
    label: "Monthly Recurring Revenue"
    description: "Ingreso recurrente mensual activo al cierre del periodo."
    synonyms: ["ingreso recurrente", "mrr", "recurrente mensual"]
    grain: month
    source_view: ceo_revenue_summary
    measure: { column: mrr, agg: sum }
    dimensions: [month, plan_name, customer_segment]
    filters_allowed: [plan_name, customer_segment, status]
    time_column: month
    format: currency_usd
    default_chart: line

  - name: mrr_growth
    label: "Crecimiento de MRR"
    description: "Variacion porcentual de MRR respecto al periodo anterior."
    synonyms: ["crecimiento mrr", "variacion de mrr"]
    grain: month
    source_view: ceo_revenue_summary
    measure: { expr: "(mrr - lag_mrr) / nullif(lag_mrr, 0)" }
    dimensions: [month]
    filters_allowed: [plan_name]
    time_column: month
    format: percent
    default_chart: line

  - name: churn_rate
    label: "Churn Rate"
    description: "Porcentaje de clientes/ingreso perdido en el periodo."
    synonyms: ["churn", "tasa de cancelacion", "fuga de clientes"]
    grain: month
    source_view: ceo_customer_health
    measure: { column: churn_rate, agg: avg }
    dimensions: [month, customer_segment]
    filters_allowed: [customer_segment]
    time_column: month
    format: percent
    default_chart: line

  - name: project_margin
    label: "Margen por Proyecto"
    description: "Margen = (ingreso reconocido - costo estimado) / ingreso reconocido."
    synonyms: ["margen de proyecto", "rentabilidad de proyecto"]
    grain: project
    source_view: ceo_project_margin
    measure: { column: margin_pct }
    dimensions: [project_name, customer_name, status]
    filters_allowed: [status, customer_name]
    format: percent
    default_chart: bar

  - name: runway_months
    label: "Runway (meses)"
    description: "Meses de operacion restantes = caja / burn rate mensual."
    synonyms: ["runway", "meses de caja", "pista financiera"]
    grain: snapshot
    source_view: ceo_financial_runway
    measure: { column: runway_months }
    dimensions: []
    filters_allowed: []
    format: number
    default_chart: kpi
```

El catalogo cubre las metricas oficiales del MVP (`docs/discovery/open-questions.md`):
MRR, ARR, crecimiento MRR, expansion revenue, churn, clientes en riesgo, pipeline por
etapa, forecast, proyectos en riesgo, margen por proyecto, tickets criticos, SLA, burn
rate, runway, costos por area. Cada metrica mapea a una view `ceo_*` ya prevista.

Campos por metrica: `name`, `label`, `description`, `synonyms`, `grain`,
`source_view`, `measure` (columna+agg o `expr`), `dimensions`, `filters_allowed`,
`time_column`, `format`, `default_chart`.

### 2.2 Contrato MetricQuery

Lo que el LLM planificador produce. No es SQL: es una seleccion estructurada validada
contra el catalogo (la validacion es la allowlist natural).

```json
{
  "metric": "mrr",
  "dimensions": ["month"],
  "filters": [{ "field": "plan_name", "op": "=", "value": "Enterprise" }],
  "time_range": { "type": "relative", "last": 6, "unit": "month" },
  "compare_to": "previous_period",
  "limit": 100
}
```

Reglas de validacion antes de compilar SQL:

- `metric` existe en el catalogo y esta permitida para el rol.
- cada `dimension` esta en `dimensions` de la metrica.
- cada `filters[].field` esta en `filters_allowed`.
- `limit` no excede el maximo global; se fuerza un `LIMIT` si falta.
- si algo no valida, el sistema pide aclaracion o cae al fallback; nunca inventa.

### 2.3 Generador SQL determinista

Funcion pura: `MetricQuery + catalogo -> SQL`. El LLM nunca escribe SQL en el camino
semantico. Para el ejemplo anterior generaria algo como:

```sql
SELECT month, SUM(mrr) AS mrr
FROM ceo_revenue_summary
WHERE plan_name = $1
  AND month >= date_trunc('month', now()) - interval '6 months'
GROUP BY month
ORDER BY month
LIMIT 100;
```

El SQL generado todavia pasa por el SQL Safety Layer (AST, solo `SELECT`, allowlist,
`LIMIT`, timeout, max rows) y se ejecuta con el rol read-only. La auditoria guarda
`path = semantic`, el `MetricQuery`, el SQL generado y validado, y el `trace_id`.

### 2.4 Fallback Text-to-SQL (gobernado)

Para preguntas exploratorias fuera del catalogo, el orquestador genera SQL candidato
como en la propuesta original, pero el camino se marca `path = fallback_sql` en
auditoria y atraviesa exactamente las mismas barreras (SQL Safety Layer + read-only).
Se mide cuantas preguntas caen aqui; las recurrentes se promueven a metricas del
catalogo.

---

## 3. Estrategia de entrega de schema al LLM

Principio: **nunca volcar el DDL crudo completo**. Se entregan tres capas, de barata a
cara, y se prefiere la mas barata que responda la pregunta.

1. **Catalogo de metricas compacto en el system prompt (cacheable).**
   El catalogo de metricas permitidas para el rol se serializa de forma compacta
   (nombre, label, descripcion corta, dimensiones, filtros) y se envia como contexto
   estable. Al ser estable entre interacciones, se beneficia de **prompt caching**: el
   costo de esos tokens se amortiza en hits de cache. Es la fuente principal del LLM
   planificador. Tamano estimado: ver supuestos de la calculadora (`docs/cost/`).

2. **Recuperacion selectiva (RAG) cuando el catalogo crezca.**
   Si el catalogo supera lo que conviene mantener siempre en contexto, se indexan las
   definiciones/sinonimos como embeddings y se recuperan las top-k metricas relevantes
   a la pregunta (patron Denodo/BIRD: vectorizar metadata y recuperar por similitud).
   Para el MVP (catalogo pequeno) probablemente no haga falta; queda como evolucion.

3. **Detalle on-demand.**
   Para precisar columnas o sinonimos puntuales, el orquestador consulta
   `GET /api/schema/catalog` o la tool MCP `describe_business_schema`, que devuelven el
   catalogo permitido por rol, no el DDL crudo.

Esta estrategia es la que mas mueve el costo: al enviar un catalogo compacto y
cacheable en vez del schema completo, los tokens de entrada por interaccion bajan
sustancialmente (ver hoja `Supuestos` de la calculadora).

---

## 4. Estrategia de modelos

Arquitectura **agnostica de proveedor**: el modelo se elige por configuracion
(`LLM_PROVIDER`, `ORCHESTRATOR_MODEL`, `LIGHT_MODEL`) y es intercambiable. Se compara
costo/calidad en `docs/cost/llm-cost-calculator.xlsx`.

### 4.1 Routing de dos niveles

No todas las interacciones necesitan el modelo mas caro. Separamos por dificultad:

| Nivel | Tareas | Perfil de modelo | Default (alternativas) |
| --- | --- | --- | --- |
| Ligero | clasificacion de intencion, deteccion de ambiguedad/aclaracion, generacion de `chart_spec`, preguntas sugeridas, ediciones visuales del mini-chat de grafica | barato, rapido, baja latencia | **GPT-5 mini** (Claude Haiku 4.5 / Gemini 2.5 Flash) |
| Planificador | NL -> `MetricQuery`, narrativa ejecutiva, analisis causal, fallback Text-to-SQL | fuerte en razonamiento estructurado | **GPT-5.2** (Claude Sonnet 4.6 / Gemini 2.5 Pro) |

Por que funciona y por que abarata: la tarea cara (elegir metrica+dimension) ya no es
"escribir SQL libre" sino "seleccionar de un catalogo", que es mas facil. Eso permite
que incluso el nivel planificador use un modelo de gama media sin perder exactitud, y
que el grueso de las interacciones (clasificacion, chart_spec, sugerencias) lo resuelva
el nivel ligero, mucho mas barato.

### 4.2 Decision de modelos (cruzando benchmark + calculadora)

Precios reales (USD por 1M de tokens, recopilados jun-2026; fuentes en la hoja
`Pricing` y abajo) y exactitud del benchmark dbt LLM SL 2026:

| Modelo | Input | Output | Cached | Exactitud capa semantica |
| --- | --- | --- | --- | --- |
| GPT-5.2 (OpenAI) | $1.75 | $14 | $0.175 | familia GPT-5.x lidera (GPT-5.3 Codex: 100%) |
| GPT-5 mini (OpenAI) | $0.25 | $2 | $0.025 | nivel ligero (tareas acotadas) |
| Claude Sonnet 4.6 | $3 | $15 | $0.30 | 98.2% |
| Claude Haiku 4.5 | $1 | $5 | $0.10 | nivel ligero |
| Gemini 2.5 Pro | $1.25 | $10 | $0.3125 | no nombrado en el benchmark |
| Gemini 2.5 Flash | $0.30 | $2.50 | $0.075 | nivel ligero |

Costo mensual por par segun la calculadora (defaults: 1 CEO, 20 interacciones/dia,
22 dias, 50% planificador, 80% cache hit):

| Par | $/interaccion | 1 usuario/mes | 100 usuarios/mes |
| --- | --- | --- | --- |
| OpenAI: GPT-5.2 + GPT-5 mini | ~$0.0075 | ~$3.31 | ~$331 |
| Claude: Sonnet 4.6 + Haiku 4.5 | ~$0.0102 | ~$4.49 | ~$449 |
| Gemini: 2.5 Pro + 2.5 Flash | ~$0.0059 | ~$2.59 | ~$259 |

**Decision: GPT-5.2 (planificador) + GPT-5 mini (ligero).**

- **Planificador GPT-5.2.** La familia GPT-5.x lidera el benchmark de capa semantica
  (GPT-5.3 Codex = 100% en datos modelados vs 84% en text-to-SQL crudo); GPT-5.2 es el
  miembro full-size costo-eficiente de esa familia. Corre ~22% mas barato por
  interaccion que Claude Sonnet 4.6 y tiene la tarifa de input cacheado mas baja
  ($0.175), lo que importa porque la entrega de schema (seccion 3) se apoya en prompt
  caching. Como el camino semantico reduce la tarea a "elegir metrica+dimension", la
  exactitud se sostiene incluso sin el variante Codex.
- **Ligero GPT-5 mini.** ~3x mas barato que Claude Haiku para clasificacion,
  aclaraciones, `chart_spec` y sugerencias, con calidad suficiente; mismo proveedor que
  el planificador (un SDK, un billing, caching consistente).
- **Embeddings (si se activa RAG): modelo de embeddings barato del proveedor elegido.**

**Matices honestos:** Gemini 2.5 Pro + Flash es el par mas barato (~$259/mes a 100
usuarios) pero no aparece en los resultados nombrados del benchmark de capa semantica,
asi que su exactitud esta menos validada; es la opcion si el presupuesto manda y se
corre una eval primero. Claude Sonnet/Haiku es la alternativa de mayor exactitud
publicada no-Codex, a mayor costo. Todo es intercambiable por configuracion
(`ORCHESTRATOR_MODEL`, `LIGHT_MODEL`) sin tocar la arquitectura.

### 4.3 Matriz comparativa (insumo de la calculadora)

| Criterio | Nivel ligero (GPT-5 mini) | Nivel planificador (GPT-5.2) |
| --- | --- | --- |
| Prioridad | costo y latencia | exactitud del `MetricQuery` y narrativa |
| Tolerancia a error | media (tareas acotadas) | baja (sostiene las cifras) |
| Volumen de interacciones | alto | medio |
| Sensible a prompt caching | si (system prompt estable) | si (catalogo de metricas) |

Los precios por proveedor y los supuestos de tokens (catalogo cacheable, system prompt,
historial, mensaje, salida) se parametrizan en `docs/cost/llm-cost-calculator.xlsx`
para calcular el costo por N usuarios e interacciones.

### Fuentes de precios

- Anthropic (Sonnet 4.6, Haiku 4.5, Opus 4.7): https://benchlm.ai/blog/posts/claude-api-pricing
  (abr-2026); oficial: https://platform.claude.com/docs/en/about-claude/pricing
- OpenAI (GPT-5.4/5.2/5.1, mini, nano): https://benchlm.ai/blog/posts/openai-api-pricing
  (2026-04-13); GPT-5 mini: https://openrouter.ai/openai/gpt-5-mini
- Google Gemini 2.5 Pro/Flash: https://ai.google.dev/gemini-api/docs/pricing ;
  https://www.tldl.io/resources/google-gemini-api-pricing
- Benchmark de exactitud: https://dbt-labs.github.io/dbt-llm-sl-bench/ ;
  https://docs.getdbt.com/blog/semantic-layer-vs-text-to-sql-2026

---

## 5. Resumen de decisiones

- Capa de metricas headless propia; el LLM emite `MetricQuery`, un generador
  determinista compila SQL sobre views `ceo_*`.
- Fallback Text-to-SQL gobernado para lo exploratorio; ambos caminos pasan por SQL
  Safety Layer y rol read-only.
- Schema al LLM en tres capas (catalogo compacto cacheable -> RAG -> detalle on-demand);
  nunca DDL crudo completo.
- Modelos por routing de dos niveles, agnostico de proveedor, **default GPT-5.2
  (planificador) + GPT-5 mini (ligero)**, decidido cruzando exactitud (benchmark) y
  costo (calculadora) e intercambiable por configuracion.

## Referencias

- ADR-0006 (`docs/adr/0006-adopt-headless-semantic-metrics-layer-and-model-strategy.md`)
- `docs/architecture/proposal.md`
- `docs/cost/README.md` y `docs/cost/llm-cost-calculator.xlsx`
- dbt LLM Semantic Layer Benchmark: https://dbt-labs.github.io/dbt-llm-sl-bench/
- dbt: Semantic Layer vs Text-to-SQL (2026):
  https://docs.getdbt.com/blog/semantic-layer-vs-text-to-sql-2026
- Denodo semantic layer for text-to-SQL:
  https://www.datamanagementblog.com/improving-the-accuracy-of-llm-based-text-to-sql-generation-with-a-semantic-layer-in-the-denodo-platform/
- Headless vs native semantic layer:
  https://venturebeat.com/ai/headless-vs-native-semantic-layer-the-architectural-key-to-unlocking-90-text
- BIRD-bench: https://beehive-advisors.com/blog/bird-bench
- Text-to-SQL con cargas complejas: https://arxiv.org/html/2407.19517v1
