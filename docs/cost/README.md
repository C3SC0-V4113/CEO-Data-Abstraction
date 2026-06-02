# Calculadora de Costos LLM

`llm-cost-calculator.xlsx` estima el costo en tokens del pipeline por N usuarios e
interacciones. Es la herramienta que pidio el negocio para decidir modelo por costo.
Soporta ADR-0006 y `docs/architecture/semantic-layer-and-model-strategy.md`.

## Como usarla

1. Abrir el `.xlsx` en Excel o LibreOffice (recalcula al abrir).
2. Editar solo las **celdas amarillas** (inputs). El resto recalcula solo.
3. Empezar por la hoja `Supuestos`, luego leer `Calculo` y `Escenarios`.

## Hojas

- **Pricing**: precio por 1,000,000 de tokens (input, output, input cacheado) por
  modelo/proveedor, con **columna Fuente** y un **bloque de fuentes (URLs + fecha)**
  debajo de la tabla. Son precios reales recopilados en jun-2026; confirmar el precio
  vigente antes de comprometer presupuesto. Esta tabla alimenta los `VLOOKUP`.
- **Supuestos**: inputs configurables.
  - Volumen: N usuarios, interacciones/usuario/dia, dias/mes, % de interacciones que
    usan el nivel planificador, % de hit de prompt caching.
  - Modelos seleccionados: planificador y ligero (dropdown que referencia `Pricing`).
  - Perfil de tokens por interaccion (planificador y ligero): contexto cacheable
    (catalogo + system prompt), entrada dinamica (historial + mensaje) y salida.
- **Calculo**: resuelve precios por lookup, calcula costo por interaccion por nivel y
  el costo diario / mensual / anual / por usuario para el modelo seleccionado.
- **Escenarios**: compara OpenAI (recomendado) vs Claude vs Gemini para 1 / 10 / 100
  usuarios, con los mismos supuestos de tokens. Edita los pares de modelo si quieres otros.

## Decision de modelos (a partir de la calculadora + benchmark)

Con los supuestos por defecto (1 CEO, 20 interacciones/dia, 22 dias, 50% al
planificador, 80% de cache hit), la hoja `Escenarios` da estos costos mensuales:

| Par (planificador + ligero) | $/interaccion | 1 usuario/mes | 100 usuarios/mes | 100 usuarios/ano |
| --- | --- | --- | --- | --- |
| OpenAI: GPT-5.2 + GPT-5 mini | ~$0.0075 | ~$3.31 | ~$331 | ~$3,973 |
| Claude: Sonnet 4.6 + Haiku 4.5 | ~$0.0102 | ~$4.49 | ~$449 | ~$5,391 |
| Gemini: 2.5 Pro + 2.5 Flash | ~$0.0059 | ~$2.59 | ~$259 | ~$3,106 |

**Decision recomendada: GPT-5.2 (heavy / planificador) + GPT-5 mini (light).**

Por que, cruzando costo (calculadora) y exactitud (benchmark dbt LLM SL):

- **Exactitud probada.** La familia GPT-5.x lidera el benchmark de capa semantica:
  GPT-5.3 Codex logra 100% (vs 84% en text-to-SQL crudo); Claude Sonnet 4.6 logra
  98.2%. GPT-5.2 es el miembro full-size costo-eficiente de esa familia lider.
- **Costo medio y muy bueno en cache.** GPT-5.2 ($1.75/$14, cached $0.175) corre
  ~22% mas barato por interaccion que Claude Sonnet 4.6 y tiene la tarifa de input
  cacheado mas baja entre los modelos fuertes, lo que importa porque nuestra estrategia
  de schema se apoya en prompt caching (catalogo de metricas cacheable).
- **Ligero costo-eficiente.** GPT-5 mini ($0.25/$2) es ~3x mas barato que Claude Haiku
  para clasificacion, aclaraciones, `chart_spec` y sugerencias, con calidad suficiente.
- **Un solo proveedor.** Mantener ambos niveles en OpenAI simplifica SDK, billing y el
  comportamiento de caching, sin perder la capa de modelos intercambiable por config.

**Matices honestos:**

- **Gemini 2.5 Pro + Flash es el par mas barato** (~$259/mes a 100 usuarios). No quedo
  como default solo porque Gemini no aparece en los resultados nombrados del benchmark
  de capa semantica, asi que su exactitud en esta tarea esta menos validada. Es la
  opcion a elegir si el presupuesto manda y se corre primero una eval de exactitud.
- **Claude Sonnet 4.6 + Haiku 4.5** es la alternativa con la mayor exactitud publicada
  en capa semantica entre modelos no-Codex (98.2%), a mayor costo.
- La decision es por configuracion (`ORCHESTRATOR_MODEL`, `LIGHT_MODEL`): cambiar de
  par no toca la arquitectura. Reevaluar cuando cambien precios o salgan modelos nuevos.

## De donde salen los supuestos de tokens

El costo lo mueve sobre todo la **entrega de schema al LLM**
(`docs/architecture/semantic-layer-and-model-strategy.md`, seccion 3): se envia un
catalogo de metricas compacto y **cacheable** (prompt caching) en vez del DDL crudo.
Por eso el perfil de tokens separa el contexto cacheable de la entrada dinamica: el
contexto cacheable se paga al precio de cache en la fraccion de hits, y eso es lo que
abarata las interacciones a volumen.

## Limitaciones

- Los precios reales son de jun-2026 (ver columna Fuente y bloque de fuentes en la hoja
  `Pricing`); cambian seguido, asi que reconfirmar antes de presupuestar.
- Asume un promedio de tokens por interaccion; preguntas muy pesadas o con muchas
  filas pueden costar mas. Ajusta el perfil de tokens para escenarios conservadores.
- No incluye costo de embeddings/RAG (solo se activa si el catalogo crece) ni de
  infraestructura (API Gateways Cloudflare, servicio MCP, backend, PostgreSQL); esos son
  costos de plataforma aparte del consumo de tokens LLM. Agrega filas si tu despliegue
  los necesita.
