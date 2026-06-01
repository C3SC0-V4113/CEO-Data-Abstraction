# Problem Framing

## Redaccion Original del Ejercicio

El ejercicio plantea que a muchas organizaciones les duele "hablar via chat con
la data". Los ejemplos son preguntas de negocio:

- Como salieron mis ventas este mes?
- Cual es mi mejor vendedor?
- Me puedes dar un pronostico?
- Dame una comparativa de mis ventas del mes pasado vs el presente.

Tambien pide considerar que ya existe un medio para chatear, no necesariamente
OpenClaw, y que la data debe consumirse por Codex o Claude via MCP.

## Reinterpretacion

El problema no es solamente conectar un chat a una base de datos. El problema es
crear una experiencia confiable para que usuarios de negocio puedan obtener
respuestas, explicaciones y acciones desde sus datos sin tener que dominar SQL,
prompt engineering o la estructura interna de la base.

## Objetivo Real

Reducir la friccion entre una pregunta de negocio y una respuesta accionable.
Esto puede lograrse con chat, pero tambien con reportes proactivos, metricas
predefinidas, preguntas sugeridas, alertas, dashboards guiados, RAG y agentes
especializados.

## Caso Base

Se usara Ventas CRM como dominio de trabajo:

- Ventas por periodo.
- Ranking de vendedores.
- Cumplimiento de metas.
- Comparativas mes contra mes.
- Forecast de ventas.
- Rendimiento y alertas por vendedor, zona o producto.

## Principio de Diseno

El LLM no deberia consultar la base con SQL libre. Debe operar sobre herramientas
controladas, metricas definidas y conocimiento de negocio verificable.
