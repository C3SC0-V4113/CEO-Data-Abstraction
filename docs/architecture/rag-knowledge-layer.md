# Capa de Conocimiento (RAG) y Orquestacion Multi-Intencion

Diseno de detalle de la capa de recuperacion documental (RAG) y de como el LLM
Orchestrator atiende prompts que mezclan metricas y conocimiento en una sola respuesta.
La decision esta en ADR-0008; la Capa Semantica de metricas (hermana de esta) esta en
`docs/architecture/semantic-layer-and-model-strategy.md` y ADR-0006.

## 1. Objetivo y alcance

El CEO debe poder, en el mismo chat, pedir:

- **Metricas de negocio** (camino existente): MRR, churn, pipeline, margen, etc., via
  `MetricQuery` -> SQL determinista sobre views `ceo_*`.
- **Informacion documental no-metrica** (camino nuevo): vision, mision, a que se dedica
  la empresa, politicas, procesos, descripciones de producto. Esa informacion vive en
  documentos heterogeneos (PDF, Word, presentaciones, Markdown, etc.).

Y, sobre todo, debe poder pedir **ambas en un solo mensaje** ("dame un grafico de MRR de
los ultimos 6 meses y recuerdame la mision de la empresa") y recibir **una sola respuesta**
que combine el artefacto de metrica y la narrativa documental con citas.

Fuera de alcance del MVP: edicion de documentos desde el chat, OCR avanzado de imagenes
escaneadas (se documenta como evolucion), y multi-tenant de conocimiento (el `access_scope`
queda modelado pero el MVP solo tiene rol CEO).

## 2. Componentes

### 2.1 Capa de Conocimiento (Knowledge / RAG Layer)

Capa hermana de la Capa Semantica. No produce SQL ni toca las views `ceo_*`: recupera
fragmentos (`chunks`) de documentos corporativos relevantes a una sub-pregunta y los
entrega al paso de sintesis para fundamentar la respuesta, citando la fuente.

### 2.2 Pipeline de Ingesta automatizada (`mirador-ingestion`)

Servicio asincrono independiente (Railway), separado del hot path porque su carga es
distinta (parseo pesado, batch, picos al subir documentos).

**Plataformas y comunicacion (decision):** el almacenamiento bruto de documentos vive en
**Cloudflare R2**, pero la **orquestacion (cola/trigger) vive en Railway**. R2 expone una
**API S3-compatible por HTTPS**, asi que `mirador-ingestion` (Node) lee/escribe R2 con el
SDK de S3 **sin bindings de Cloudflare** (a diferencia de Vectorize/D1). No se usan
Cloudflare Queues: la cola es interna de Railway (tabla de jobs en PostgreSQL o Redis), para
no depender de un Worker-puente. Los embeddings van a `pgvector` en el mismo Postgres.

Flujo:

1. **Aterrizaje:** el archivo se sube a R2. La subida pasa por `mirador-ingestion` (o por
   una URL prefirmada de R2) y el servicio **encola un job internamente** (Railway). Cada
   objeto lleva metadata minima (titulo, tipo, `access_scope`).
2. **Trigger:** el worker de Railway consume su propia cola y procesa el job. La ingesta es
   **automatica**: no requiere accion manual por documento. (No se usan Cloudflare Queues.)
3. **Parseo por formato:** extraccion de texto segun el tipo (PDF, docx, pptx, xlsx, txt,
   md, html). Cada parser normaliza a texto plano + estructura (paginas/secciones).
4. **Normalizacion:** limpieza, deduplicado de espacios, conservacion de encabezados y
   numero de pagina/seccion para poder citar.
5. **Chunking:** division en fragmentos con solapamiento (overlap) respetando limites
   semanticos/estructurales (titulos, parrafos). Tamano de chunk acotado.
6. **Embedding:** se calcula el vector de cada chunk con el `EMBEDDING_MODEL` configurado.
   Se hace **una sola vez por chunk** (no en cada consulta).
7. **Upsert:** se insertan/actualizan `documents` y `document_chunks` en pgvector. La
   idempotencia se garantiza con `content_hash` por documento y por chunk: si el contenido
   no cambio, no se reembebe.
8. **Re-index / borrado:** al reemplazar un documento se reindexan sus chunks; al eliminar
   el objeto se borran sus chunks y su registro.

### 2.3 Knowledge Retrieval (modulo interno de `mirador-core`)

Vive dentro del backend Fastify, en el hot path. Para una sub-pregunta de tipo
`knowledge_lookup`:

1. Embebe la consulta con el mismo `EMBEDDING_MODEL`.
2. Ejecuta busqueda vectorial **top-k** en `document_chunks` via Prisma, con rol
   **read-only** y filtrada por `access_scope`/rol.
3. Opcional: **rerank** de los candidatos (off por defecto; configurable).
4. Devuelve los chunks con su metadata de cita (documento, pagina/seccion, version).

### 2.4 `KnowledgeBaseContext`

Resumen **compacto y cacheable** de los temas/documentos disponibles (titulos,
descripciones cortas, tags, `access_scope`), analogo a `MetricCatalogContext`. Se entrega
al planner para que **sepa cuando existe conocimiento documental relevante** y enrute a
`knowledge_lookup` sin tener que recuperar nada todavia. No incluye el contenido de los
documentos, solo el indice gobernado.

### 2.5 Registro de documentos (Knowledge Catalog)

Tabla `documents`: metadata, version, `content_hash`, `access_scope`, estado de
indexacion. Es la gobernanza analoga al catalogo de metricas: define que documentos son
"oficiales" y consultables, y permite auditar que se indexo.

## 3. Orquestacion multi-intencion

El LLM Orchestrator ya construye un `output_plan` combinando el `intent_mode` del composer
con los requisitos explicitos del prompt. Se **extiende** ese concepto a un **plan de
ejecucion tipado**.

### 3.1 Planificacion (modelo ligero, 1 llamada)

El modelo ligero recibe el prompt + `MetricCatalogContext` + `KnowledgeBaseContext` (ambos
cacheables) y devuelve un `execution_plan`: lista de sub-tareas tipadas.

```json
{
  "execution_plan": [
    {
      "id": "t1",
      "type": "metric_query",
      "sub_question": "MRR de los ultimos 6 meses",
      "output": "chart"
    },
    {
      "id": "t2",
      "type": "knowledge_lookup",
      "sub_question": "mision de la empresa"
    }
  ],
  "output_plan": ["chart", "text"]
}
```

Tipos de sub-tarea:

- `metric_query`: va a la Capa Semantica (MetricQuery -> SQL determinista).
- `knowledge_lookup`: va al Knowledge Retrieval (RAG).
- `direct_answer`: el modelo responde directo (saludos, aclaraciones, meta-preguntas) sin
  metrica ni recuperacion.

El `execution_plan` se **valida** (tipos permitidos, limites de sub-tareas) y se **audita**.

### 3.2 Despacho en paralelo

Las sub-tareas se ejecutan **concurrentemente** (`Promise.all`) para minimizar latencia:

- `metric_query` -> `MetricCatalogContext` -> `MetricQuery` -> validacion -> SQL
  determinista -> **SQL Safety Layer** -> PostgreSQL read-only. (Si esta fuera del
  catalogo, cae al fallback Text-to-SQL auditado, igual que hoy.)
- `knowledge_lookup` -> Knowledge Retrieval -> top-k chunks con metadata de cita.

Cada camino conserva sus barreras: el camino de metrica no cambia; el de conocimiento es
read-only y filtrado por `access_scope`.

### 3.3 Sintesis (modelo planificador, 1 llamada)

Un unico pase del modelo planificador recibe los resultados de todas las sub-tareas y
compone **una sola respuesta**:

- Artefactos de metrica (chart/table/kpi) generados por el camino semantico.
- Narrativa ejecutiva fundamentada en los chunks recuperados, con **citas** a documentos
  fuente (`citations`/`sources`).
- Warnings y `suggested_questions`.

El system prompt de sintesis esta **endurecido**: el contenido de los documentos se trata
como **dato, no como instruccion**; si no hay evidencia documental suficiente, la respuesta
debe decir que no se encontro en la base de conocimiento en vez de inventar.

### 3.4 Ejemplo de respuesta combinada

```json
{
  "answer": "El MRR crecio 8% en los ultimos 6 meses (ver grafica). Sobre la mision: ...",
  "artifacts": [
    { "artifact_type": "chart", "source_views": ["ceo_revenue_summary"] }
  ],
  "citations": [
    {
      "document_id": "doc_uuid",
      "title": "Manifiesto de la empresa 2026",
      "locator": "pag. 2",
      "snippet": "Nuestra mision es..."
    }
  ],
  "warnings": [],
  "metadata": { "execution_plan": ["metric_query", "knowledge_lookup"] },
  "trace_id": "uuid"
}
```

## 4. Modelo de datos (pgvector)

Se agregan dos tablas al PostgreSQL existente (ver `docs/diagrams/mermaid/database-model.mmd`):

- **`documents`**: `id`, `title`, `source_uri` (R2), `doc_type`, `version`, `content_hash`,
  `access_scope`, `indexed_at`, `status`.
- **`document_chunks`**: `id`, `document_id`, `chunk_index`, `content`, `embedding`
  (`vector`), `token_count`, `locator` (pagina/seccion), `content_hash`.

Indexacion vectorial con `ivfflat` o `hnsw` segun volumen. El catalogo de metricas sigue
siendo configuracion YAML/JSON, no tabla; el knowledge catalog si es tabla (`documents`)
porque su contenido es dinamico.

`query_audit_log` se extiende con `execution_plan` y `retrieved_doc_ids` para trazar que
sub-tareas se ejecutaron y que documentos fundamentaron la respuesta.

## 5. Estrategia de modelos y embeddings

Se reutiliza el **routing de dos niveles** (ver
`docs/architecture/semantic-layer-and-model-strategy.md`):

- **Nivel ligero** (`LIGHT_MODEL`): planificacion/descomposicion del `execution_plan`,
  clasificacion, aclaraciones, `chart_spec`.
- **Nivel planificador** (`ORCHESTRATOR_MODEL`): `MetricQuery`, fallback SQL y la
  **sintesis** final con citas.

Se agrega un **tercer eje de configuracion**, agnostico de proveedor:

- `EMBEDDING_PROVIDER`, `EMBEDDING_MODEL`: modelo de embeddings para ingesta y para la
  query. Debe ser el **mismo** en ingesta y recuperacion (misma dimensionalidad/espacio).

**Modelo de embeddings definido: `text-embedding-3-small` (OpenAI)** como default, con
`text-embedding-3-large` como upgrade de calidad. Criterio: mismo proveedor que la
generacion (un solo SDK/billing/caching). La calculadora (hoja `RAG`) muestra que el costo
RAG casi no depende del modelo de embeddings —la palanca es la sintesis—, asi que gana la
coherencia. Alternativas comparadas (precio, dimensiones, MTEB, multilingue): `voyage-3` /
`voyage-3-lite` (Voyage; Anthropic recomienda Voyage, no tiene embeddings propios), Cohere
`embed v3`, `gemini-embedding-001` (Google) y `bge-m3` (Workers AI / self-host, con salto
cross-cloud). Cambiar de modelo obliga a **reindexar**. Detalle y comparacion en
`docs/cost/architecture-cost.md` y la hoja `RAG` de `llm-cost-calculator.xlsx`.

## 6. Optimizacion de costo (buenas practicas agenticas)

- **Routing por nivel:** planificacion y tareas acotadas con el modelo ligero; el
  planificador solo para `MetricQuery` y sintesis.
- **Prompt caching:** system prompt + definiciones de tools + `MetricCatalogContext` +
  `KnowledgeBaseContext` son estables entre interacciones y se cachean.
- **Recuperacion condicional:** si el `execution_plan` no tiene `knowledge_lookup`, no se
  embebe ni se recupera nada (cero costo de RAG). Lo mismo para el camino de metrica.
- **Acotamiento:** top-k y tamano de chunk limitados; rerank opcional.
- **Embeddings de documentos one-time:** se calculan en ingesta, no por consulta;
  idempotencia por `content_hash` evita reembeber.

El detalle numerico (costo de plataforma por servicio + costo de agentes desde la
calculadora + filas de embeddings/RAG) esta en `docs/cost/architecture-cost.md`.

## 7. Gobernanza y seguridad

- **Read-only:** la recuperacion usa el rol read-only de Postgres, igual que las metricas.
- **`access_scope` por documento:** la recuperacion filtra por rol; el MVP (solo CEO) ve
  todo, pero el modelo no se acopla a esa suposicion.
- **Contenido como dato, no instruccion:** mitigacion de prompt-injection en la sintesis;
  sanitizacion del contenido recuperado antes de inyectarlo.
- **Citas obligatorias:** la narrativa documental cita documento + localizador; sin
  evidencia, se responde que no se encontro.
- **Auditoria extendida:** `execution_plan`, `retrieved_doc_ids` y conteo de chunks en
  `query_audit_log`, ligados al `trace_id`.

## 8. Servicios y repositorios

| Repo / servicio | Rol relativo a RAG | Alojamiento |
| --- | --- | --- |
| `mirador-core` | LLM Orchestrator + planner + **Knowledge Retrieval** | Railway |
| `mirador-ingestion` | **Pipeline de ingesta** (parseo, chunking, embeddings, upsert) | Railway |
| `mirador-web` | Renderiza respuesta combinada (artefactos + citas) | Cloudflare Workers |
| `mirador-mcp` | Expone el conocimiento via tool MCP a la Core Internal API | Railway |

Infra compartida (no es repo): PostgreSQL + `pgvector` (Railway) y bucket de documentos
(Cloudflare R2).

## 9. Referencias

- ADR-0008 (decision de esta capa).
- `docs/architecture/proposal.md` (arquitectura maestra).
- `docs/architecture/semantic-layer-and-model-strategy.md` (capa hermana de metricas).
- `docs/cost/architecture-cost.md` y `docs/cost/README.md`.
- Diagramas: `docs/diagrams/mermaid/rag-ingestion-flow.mmd`,
  `docs/diagrams/mermaid/rag-multi-intent-orchestration.mmd`.
