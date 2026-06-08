# ADR-0008: Adoptar Capa de Conocimiento (RAG) y Orquestacion Multi-Intencion

## Status

Proposed

## Date

2026-06-08

## Context

El chatbot ejecutivo hoy solo responde **metricas de negocio**. El LLM Orchestrator
produce un `MetricQuery` validado contra el catalogo (camino semantico) o cae al
fallback Text-to-SQL, siempre sobre views `ceo_*` y bajo el SQL Safety Layer con rol
read-only (ver `docs/architecture/proposal.md`, ADR-0006 y ADR-0007).

Surge un requisito nuevo: dentro del **mismo chat**, el CEO tambien debe poder pedir
informacion **no-metrica y documental** ("dame la vision de la empresa", "a que se
dedica la empresa", politicas, procesos). Esos contenidos llegan en archivos
heterogeneos (PDF, Word, presentaciones, etc.) y su alojamiento es libre. Ademas, un
solo prompt puede pedir **dos cosas a la vez** (por ejemplo "un grafico de MRR y la
mision de la empresa") y **ambas deben venir en una sola respuesta**. Esto obliga a un
componente que lea el prompt, lo **descomponga** y **enrute** cada parte a la capa
correcta (Capa Semantica/SQL vs. recuperacion documental).

RAG quedo marcado como fuera de alcance en ADR-0006 y en el glosario. Este ADR lo
**incorpora formalmente** como una capa hermana de la Capa Semantica, sin romper las
barreras de seguridad ya decididas.

Restriccion tecnica relevante: el backend Fastify corre en Railway (Node), por lo que
no puede usar bindings de Cloudflare (D1/Vectorize solo viven dentro de un Worker).
Cualquier opcion debe ser compatible con el acceso desde Fastify/Prisma.

## Decision Drivers

- Responder preguntas documentales no-metricas dentro del mismo chat ejecutivo.
- Soportar prompts multi-intencion (metrica + conocimiento) en una sola respuesta.
- Conservar la gobernanza existente: rol read-only, auditoria, sin SQL libre.
- Costo acotado y predecible, siguiendo buenas practicas agenticas (routing por nivel,
  prompt caching, recuperacion condicional).
- Minimizar componentes operativos nuevos y mantener coherencia con el stack
  (Fastify + Prisma + PostgreSQL + Railway / Cloudflare).
- Compatibilidad real con Fastify en Railway (evitar depender de bindings Cloudflare).

## Considered Options

### Option 1: pgvector en el PostgreSQL existente + planner estructurado

- Pros:
  - Reusa el PostgreSQL de Railway, Prisma y el rol read-only; el vector vive junto a
    los datos, con gobernanza y auditoria unificadas.
  - Sin salto cross-cloud en el hot path de recuperacion.
  - Minima infra nueva: la recuperacion es un modulo interno de Fastify; solo la
    ingesta se separa como servicio asincrono.
  - El planner estructurado extiende el `output_plan` que ya existe; costo predecible
    y auditable.
- Cons:
  - El computo de embeddings e ingesta corre por nuestra cuenta.
  - La escala vectorial de Postgres es suficiente para el volumen documental esperado,
    pero no es "infinita" como un servicio vectorial dedicado.

### Option 2: Cloudflare-native (Vectorize + Workers AI + R2) tras un Worker

- Pros:
  - Ingesta y embeddings 100% serverless; aprovecha Cloudflare ya presente en el stack.
  - Bindings nativos dentro del Worker (resuelve la duda de compatibilidad con Fastify
    si el Worker expone un servicio HTTP).
  - Escala vectorial sin esfuerzo operativo.
- Cons:
  - Segunda plataforma de datos y salto cross-cloud en cada recuperacion (latencia y un
    dominio de fallo extra en el hot path).
  - Gobernanza y auditoria quedan en una superficie aparte del rol read-only de Postgres.
  - Embeddings atados a los modelos de Workers AI.

### Option 3: Sin RAG (status quo)

- Pros:
  - Nada nuevo que operar.
- Cons:
  - No cumple el requisito: las preguntas documentales quedan sin responder o el modelo
    las inventa sin fuente verificable.

Para la orquestacion multi-intencion se consideraron dos sub-opciones: un **planner
estructurado** (un pase de descomposicion que emite un plan tipado y se despacha de
forma determinista) frente a un **loop de tool-use nativo** (el modelo decide y encadena
tool-calls libremente). Se elige el planner estructurado por costo predecible y
auditabilidad; el loop abierto puede encarecerse y es menos predecible.

## Decision

Adoptar la **Option 1**: una **Capa de Conocimiento (RAG)** con **pgvector** en el
PostgreSQL existente, y **orquestacion multi-intencion mediante un planner estructurado**
que extiende el `output_plan` del LLM Orchestrator. El diseno completo esta en
`docs/architecture/rag-knowledge-layer.md`.

Puntos clave de la decision:

- **Vector store:** extension `pgvector` sobre el PostgreSQL de Railway; tablas
  `documents` (registro/knowledge catalog) y `document_chunks` (embeddings + metadata).
- **Recuperacion (retrieval):** modulo interno del backend (`ceo-chat-core`) que consulta
  pgvector via Prisma bajo el rol read-only.
- **Ingesta automatizada:** servicio asincrono independiente (`ceo-chat-ingestion`,
  Railway). El archivo bruto vive en **Cloudflare R2**, alcanzado por **API S3** (sin
  bindings); la **cola/trigger es interna de Railway** (jobs en PostgreSQL o Redis), no
  Cloudflare Queues. Pasos: parseo por formato, normalizacion, chunking, embeddings y
  upsert idempotente (`content_hash`). Los embeddings van a `pgvector` en Railway.
- **Orquestacion:** un pase del modelo ligero descompone el prompt en sub-tareas tipadas
  (`metric_query`, `knowledge_lookup`, `direct_answer`); se despachan en paralelo (camino
  semantico existente + recuperacion documental); un pase del modelo planificador hace la
  **sintesis** de una sola respuesta con artefactos de metrica y narrativa con **citas**.
- **Modelos:** se reusa el routing de dos niveles; se agrega un eje de configuracion para
  embeddings (`EMBEDDING_PROVIDER`, `EMBEDDING_MODEL`), agnostico de proveedor.

## Rationale

pgvector encaja mejor porque mantiene **una sola plataforma de datos y una sola
gobernanza**: la recuperacion es otra consulta read-only auditada, no una nueva frontera
de seguridad. Evita el salto cross-cloud en el hot path y resuelve la duda de
compatibilidad sin bindings (todo es Prisma/SQL desde Fastify). Es la misma logica de
"minimizar componentes y reutilizar una sola capa" que motivo ADR-0007.

El planner estructurado se prefiere al loop de tool-use abierto porque el dominio es
acotado (dos o tres caminos, sin encadenamiento profundo): un pase de planificacion con
el modelo ligero da costo predecible, el plan queda auditado y el despacho es
determinista. Cloudflare-native (Option 2) queda como alternativa valida si el volumen
documental crece mas alla de lo razonable para Postgres; la decision de vector store es
reversible porque la interfaz de recuperacion queda detras de un modulo interno.

## Consequences

### Positive

- El chat responde preguntas documentales y multi-intencion en una sola respuesta.
- Gobernanza homogenea: recuperacion read-only, auditada y con citas verificables.
- Costo acotado: recuperacion condicional, top-k limitado, embeddings de documentos
  calculados una sola vez en ingesta y contextos cacheables.
- Disponible tanto para la web como para MCP, porque la logica vive en la Core Internal
  API (se reutiliza como el resto del pipeline).

### Negative

- Nuevo servicio que operar (`ceo-chat-ingestion`) y nueva extension de base (`pgvector`).
- Mas costo de tokens por interaccion cuando hay `knowledge_lookup` (chunks recuperados).
- Costo de embeddings de ingesta (one-time por documento) y por query.

### Risks and Mitigations

- Riesgo: prompt-injection desde el contenido de los documentos.
  - Mitigacion: tratar el contenido recuperado como **dato, no como instruccion**;
    sanitizar antes de inyectar; system prompt endurecido en la sintesis.
- Riesgo: respuestas no fundamentadas o alucinadas.
  - Mitigacion: exigir **citas** a documentos; si no hay evidencia suficiente, responder
    que no se encontro en la base de conocimiento.
- Riesgo: crecimiento de tokens por chunks recuperados.
  - Mitigacion: top-k y tamano de chunk acotados; recuperacion condicional (cero costo si
    el plan no incluye `knowledge_lookup`).
- Riesgo: escala de la busqueda vectorial en Postgres.
  - Mitigacion: indices `ivfflat`/`hnsw` en pgvector; reevaluar Vectorize (Option 2) si el
    volumen lo exige.
- Riesgo: documentos con informacion sensible.
  - Mitigacion: `access_scope` por documento y filtrado por rol en la recuperacion (CEO ve
    todo en el MVP, pero el modelo no se acopla a esa suposicion).

## Implementation Notes

- Habilitar la extension `pgvector`; modelar `documents` y `document_chunks` en Prisma.
- `ceo-chat-ingestion`: archivo bruto en R2 (API S3, sin bindings); cola/trigger interna de
  Railway (jobs en PostgreSQL o Redis), no Cloudflare Queues; parseo por formato
  (PDF/docx/pptx/...); chunking con overlap; embeddings; upsert idempotente por
  `content_hash`; re-index al cambiar y borrado al eliminar.
- Knowledge Retrieval en `ceo-chat-core`: embeber query, busqueda top-k, rerank opcional
  (off por defecto), siempre read-only.
- Orchestrator: `execution_plan` tipado y validado (modelo ligero) -> despacho paralelo
  -> sintesis (modelo planificador).
- Auditoria: agregar `execution_plan` y `retrieved_doc_ids` a `query_audit_log`.

## Related Decisions

- ADR-0005 (chatbot-first como experiencia principal).
- ADR-0006 (capa semantica headless y estrategia de modelos; RAG estaba fuera de alcance).
- ADR-0007 (API Gateways y servicio MCP independiente; patron de reutilizar la Core
  Internal API).

## References

- `docs/architecture/rag-knowledge-layer.md`
- `docs/architecture/proposal.md`
- `docs/architecture/semantic-layer-and-model-strategy.md`
- `docs/cost/architecture-cost.md`
- `docs/cost/README.md`
