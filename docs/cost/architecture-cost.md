# Costo de Arquitectura por Servicio

`llm-cost-calculator.xlsx` (ver `README.md`) estima solo el **consumo de tokens LLM** del
pipeline de metricas. Este documento cubre lo que esa calculadora deja fuera: el **costo
de plataforma por servicio/alojamiento**, mas el **costo aproximado de agentes** (tomado de
la calculadora) y el **costo incremental de RAG** (embeddings e ingesta). Soporta ADR-0008.

> **Caveat:** los precios de plataforma y modelos son representativos de **jun-2026** y
> cambian seguido. Confirmar el precio vigente de cada proveedor antes de comprometer
> presupuesto, igual que en `README.md`. Las cifras buscan dar **orden de magnitud**, no un
> presupuesto cerrado.

## 1. Servicios y alojamiento

Repos/servicios de la arquitectura (ver `docs/architecture/proposal.md` y
`docs/architecture/rag-knowledge-layer.md`):

| Repo / servicio | Rol | Alojamiento |
| --- | --- | --- |
| `ceo-chat-web` | Frontend Next.js SSR | Cloudflare Workers |
| Web + MCP API Gateways | Borde (TLS, rate limit, WAF, cuotas) | Cloudflare |
| `ceo-chat-core` | Backend Fastify + Orchestrator + Semantica + Safety + Knowledge Retrieval | Railway |
| `ceo-chat-mcp` | Servicio MCP adapter | Railway |
| `ceo-chat-ingestion` | Pipeline de ingesta RAG | Railway |
| PostgreSQL + `pgvector` | Datos (metricas + vectores) | Railway |
| Bucket de documentos | Archivos fuente para RAG | Cloudflare R2 |
| Proveedor LLM + embeddings | Tokens de agentes (GPT-5.2 / GPT-5 mini) + embeddings (text-embedding-3-small) | OpenAI (default) |

## 2. Costo de plataforma (representativo, mensual)

Cifras de orden de magnitud para un MVP de bajo volumen (1 usuario CEO). Escalan
suavemente hasta decenas de usuarios porque el cuello de botella es el consumo de tokens,
no la plataforma.

| Componente | Plan / tarifa representativa | Costo mensual aprox. |
| --- | --- | --- |
| Cloudflare Workers (`ceo-chat-web`) | Workers Paid (~$5/mo) o free tier en MVP | $0 - $5 |
| Cloudflare API Gateways (WAF/rate limit) | Incluido / add-ons segun reglas | $0 - bajo |
| Railway `ceo-chat-core` | Servicio always-on (uso) | ~$5 - $10 |
| Railway `ceo-chat-mcp` | Servicio liviano (uso) | ~$5 |
| Railway `ceo-chat-ingestion` | Worker asincrono, escala a ~0 entre cargas | ~$0 - $5 |
| Railway PostgreSQL + `pgvector` | Instancia gestionada (uso + storage) | ~$5 - $15 |
| Cloudflare R2 | $0.015/GB-mes storage, sin egress fees | < $1 (docs del MVP) |
| **Subtotal plataforma** | | **~$20 - $45 / mes** |

Notas:

- Railway es **usage-based**: el costo real depende de CPU/RAM/horas; los rangos asumen
  servicios pequenos para MVP. `ceo-chat-ingestion` puede escalar casi a cero entre cargas
  de documentos.
- Los API Gateways de Cloudflare son control de trafico; el costo extra aparece con reglas
  WAF avanzadas o volumen alto, no en el MVP.
- R2 es barato y **sin cargos de egress**, ventaja frente a S3 para servir documentos.

## 3. Costo de agentes (tokens LLM) — desde la calculadora

Reusando los supuestos por defecto de `llm-cost-calculator.xlsx` (1 CEO, 20 interacciones/
dia, 22 dias, 50% al planificador, 80% de cache hit) y la hoja `Escenarios`:

| Par (planificador + ligero) | $/interaccion | 1 usuario/mes | 100 usuarios/mes |
| --- | --- | --- | --- |
| **OpenAI: GPT-5.2 + GPT-5 mini** (default) | ~$0.0075 | ~$3.31 | ~$331 |
| Claude: Sonnet 4.6 + Haiku 4.5 | ~$0.0102 | ~$4.49 | ~$449 |
| Gemini: 2.5 Pro + 2.5 Flash | ~$0.0059 | ~$2.59 | ~$259 |

Esta es la palanca dominante a volumen: a 100 usuarios el costo de tokens (~$331/mes con el
par default) supera al de plataforma. Detalle y fuentes en `README.md` y la hoja `Pricing`.

## 4. Costo incremental de RAG

Modelado en la hoja **RAG** de `llm-cost-calculator.xlsx`. **Modelo de embeddings definido:
`text-embedding-3-small` (OpenAI)** (~$0.02/1M), con `text-embedding-3-large` como upgrade.
RAG agrega tres conceptos sobre el costo de agentes:

### 4.1 Embeddings de ingesta (one-time por documento)

Se calculan **una sola vez** al indexar (idempotencia por `content_hash`). Con los defaults
de la hoja (200 docs x 8,000 tokens = 1.6M tokens) y `text-embedding-3-small`: **~$0.03
one-time**. Despreciable; cambiar el `EMBEDDING_MODEL` obliga a reindexar y repite el costo.

### 4.2 Embedding de la query (por interaccion con `knowledge_lookup`)

- ~80 tokens por consulta embebida -> fraccion de centavo. Despreciable (~$0.0003/mes a 1
  usuario).

### 4.3 Tokens extra de sintesis (chunks recuperados)

Es la **palanca dominante** de RAG: los chunks top-k entran como input al pase de sintesis
del planificador (GPT-5.2, input normal, no cacheable).

- Default: top-k = 5 x ~300 tokens = ~1,500 tokens de input extra por interaccion-RAG.
- Solo aplica cuando el `execution_plan` incluye `knowledge_lookup` (recuperacion
  condicional). Si el prompt es solo metrica, el costo RAG es **cero**.

### 4.4 Total RAG incremental (hoja RAG, defaults)

Con 40% de interacciones usando conocimiento, top-k 5:

| Usuarios | Ingesta inicial (one-time) | Total RAG incremental/mes |
| --- | --- | --- |
| 1 | ~$0.03 | ~$0.47 |
| 10 | ~$0.03 | ~$4.63 |
| 100 | ~$0.03 | ~$46.2 |

**El modelo de embeddings casi no mueve el total**: a 1 usuario, el total RAG/mes va de
~$0.46 (3-small / voyage-3-lite / bge-m3) a ~$0.49 (gemini); la diferencia esta en la
sintesis, no en el embedding. Por eso se elige `text-embedding-3-small` por coherencia de
proveedor (ver hoja `RAG`, seccion de comparacion, y `docs/cost/README.md`).

## 5. Totales aproximados

| Usuarios | Plataforma/mes | Agentes (OpenAI default)/mes | RAG incremental/mes | Total aprox./mes |
| --- | --- | --- | --- | --- |
| 1 | ~$20 - $45 | ~$3.31 | ~$0.47 | ~$25 - $50 |
| 10 | ~$25 - $55 | ~$33 | ~$4.6 | ~$65 - $95 |
| 100 | ~$40 - $80 | ~$331 | ~$46 | ~$420 - $460 |

A bajo volumen domina la **plataforma**; a 100 usuarios domina el **consumo de tokens**.
RAG agrega poco salvo que se suba mucho el top-k o el tamano de chunk.

## 6. Como mantener este documento

- La hoja **RAG** de `llm-cost-calculator.xlsx` ya modela embeddings y tokens de chunks;
  ajustar sus variables amarillas (N docs, top-k, % RAG, modelo) para recalcular escenarios.
- Reconfirmar tarifas de Railway/Cloudflare/OpenAI antes de cualquier presupuesto formal.
- Revisar al cambiar de `EMBEDDING_MODEL` (obliga a reindexar) o de par de modelos.

## 7. Referencias

- `docs/cost/README.md` y `llm-cost-calculator.xlsx`.
- ADR-0008 y `docs/architecture/rag-knowledge-layer.md`.
- `docs/architecture/semantic-layer-and-model-strategy.md`.
