# ADR-0001: Adoptar una Estrategia MCP-first para Acceso a Datos con IA

## Status

Proposed

## Date

2026-06-01

## Context

El ejercicio pide una propuesta para que las organizaciones puedan "hablar con la
data" usando chat o agentes como Codex y Claude via MCP. Los ejemplos iniciales
incluyen ventas del mes, mejor vendedor, forecast y comparativas entre periodos.

Despues del analisis inicial, el problema se entiende de forma mas amplia: no se
trata solo de habilitar un chat, sino de reducir la dependencia de que el usuario
sepa formular buenos prompts. Muchos usuarios de negocio no saben que preguntar,
repiten consultas frecuentes o necesitan que el sistema les proponga insights
antes de iniciar una conversacion.

El dominio de referencia sera Ventas CRM para mantener ejemplos concretos.

## Decision Drivers

- Reducir la necesidad de prompt engineering para usuarios de negocio.
- Evitar que un LLM ejecute SQL libre contra datos productivos.
- Permitir consumo desde agentes como Codex o Claude mediante herramientas
  controladas.
- Mantener trazabilidad sobre metricas, fuentes, filtros y resultados.
- Habilitar RAG, agentes especializados y reportes proactivos como opciones de
  brainstorming.

## Considered Options

### Option 1: Chatbot directo conectado a la base de datos

- Pros: experiencia simple de explicar, baja friccion inicial, demo rapida.
- Cons: alto riesgo si genera SQL libre, respuestas inconsistentes, depende de
  que el usuario sepa preguntar, dificil de auditar.

### Option 2: MCP-first con herramientas gobernadas

- Pros: Codex/Claude consumen herramientas explicitas, se controla que metricas
  existen, se reduce riesgo operativo, se puede auditar cada llamada.
- Cons: requiere disenar contratos de herramientas, capa semantica y permisos.

### Option 3: Dashboard + copiloto guiado

- Pros: buena experiencia para negocio, reduce prompts con botones, filtros y
  preguntas sugeridas; encaja con una futura app React.
- Cons: mayor alcance de producto, requiere UX, backend y mantenimiento visual.

### Option 4: Reportes y alertas automaticas con IA

- Pros: resuelve consultas repetitivas, entrega insights sin que el usuario
  pregunte, puede ejecutar procesos pesados fuera de horas pico.
- Cons: puede producir ruido, requiere calibrar frecuencia, umbrales y
  relevancia; no reemplaza exploracion ad hoc.

## Decision

Proponer una estrategia **MCP-first** como base de acceso a datos con IA,
manteniendo las automatizaciones, RAG, agentes especializados y experiencia
guiada como opciones complementarias de brainstorming.

Esta decision queda en estado `Proposed` porque todavia se esta explorando el
problema y no se ha cerrado la arquitectura final.

## Rationale

MCP-first ofrece un punto medio pragmatico: permite que agentes existentes
consuman datos mediante herramientas controladas, sin comprometerse de inmediato
a construir una app completa. Tambien permite evolucionar hacia dashboards,
reportes automaticos y copilotos guiados sin cambiar la base conceptual.

La clave es no exponer la base de datos de forma directa al LLM. El sistema debe
ofrecer herramientas con nombres, parametros, permisos y metricas definidas.

## Consequences

### Positive

- Mayor control sobre que datos y metricas pueden consultarse.
- Mejor auditoria de solicitudes y resultados.
- Menor riesgo que SQL libre generado por IA.
- Buen encaje con Codex, Claude y otros clientes MCP.
- Base extensible para agentes, RAG y jobs.

### Negative

- Requiere disenar una capa semantica antes de obtener respuestas confiables.
- Puede sentirse menos flexible que un chat libre si las herramientas son pocas.
- Necesita gobernanza de metricas, permisos y definiciones.

### Risks and Mitigations

- Riesgo: el sistema responde con metricas ambiguas.
  Mitigacion: glosario, capa semantica y respuestas con definicion de metrica.
- Riesgo: usuarios siguen sin saber que pedir.
  Mitigacion: catalogo de metricas, preguntas sugeridas y reportes proactivos.
- Riesgo: automatizaciones generan ruido.
  Mitigacion: tratarlas como hipotesis, medir uso y permitir configuracion por
  rol.

## Implementation Notes

- Disenar herramientas MCP allowlisted como `list_metrics`, `query_metric`,
  `compare_periods`, `rank_sellers`, `generate_forecast`, `schedule_report` y
  `get_report`.
- Usar Ventas CRM como dominio de prueba para definir tablas y metricas.
- Separar datos estructurados de conocimiento documental usado por RAG.
- Documentar cada metrica con nombre, formula, filtros permitidos, granularidad
  y fuente.

## Related Decisions

- Pendiente: decision sobre base de datos.
- Pendiente: decision sobre capa semantica.
- Pendiente: decision sobre arquitectura de RAG.
- Pendiente: decision sobre experiencia de usuario final.

## References

- `docs/discovery/problem-framing.md`
- `docs/strategy/solution-options.md`
- `docs/strategy/mcp-first-access.md`
