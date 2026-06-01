# Hablar con la Data sin Depender Siempre del Chat

Este repositorio documenta una propuesta de estrategia para abordar un problema
comun en organizaciones: permitir que las personas consulten y entiendan sus
datos con ayuda de IA, sin depender siempre de saber escribir buenos prompts.

El ejercicio original apunta a preguntas como:

- "Como salieron mis ventas este mes?"
- "Cual es mi mejor vendedor?"
- "Puedes darme un pronostico?"
- "Dame una comparativa de mis ventas del mes pasado vs el presente"

La lectura inicial es que el objetivo no deberia ser solo construir otro
chatbot. El problema mas profundo es reducir friccion para usuarios de negocio:
muchos no saben que pedir, no tienen practica en prompt engineering, repiten las
mismas consultas o necesitan respuestas accionables sin navegar una conversacion
larga.

## Enfoque

El caso base de discovery sera un dominio tipo **Ventas CRM**. Esto permite
aterrizar ejemplos con ventas mensuales, vendedores, metas, forecast,
comparativas, rendimiento y reportes ejecutivos.

La estrategia explorara varias soluciones con IA:

- MCP-first para que Codex, Claude u otros agentes consuman herramientas de data
  controladas.
- Catalogos de metricas en lenguaje natural para evitar SQL libre y prompts
  fragiles.
- Preguntas sugeridas segun contexto, rol, fechas, anomalias o cambios
  recientes.
- Reportes automaticos con IA como hipotesis de brainstorming, no como decision
  cerrada.
- Alertas inteligentes cuando una metrica cruza umbrales o cambia de forma
  anomala.
- RAG sobre conocimiento de negocio para explicar definiciones, reglas y
  resultados.
- Agentes especializados para ventas, forecast, rendimiento, explicacion
  ejecutiva y calidad de datos.
- Una posible experiencia futura tipo dashboard + copiloto, donde React puede
  aportar una interfaz guiada para usuarios de negocio.

## Estructura

```text
docs/
  adr/            Decisiones de arquitectura y plantilla ADR.
  discovery/      Problema, usuarios, casos de uso, datos asumidos y preguntas.
  glossary/       Terminos compartidos del ejercicio.
  strategy/       Opciones de solucion y lineas de brainstorming.
  architecture/   Espacio reservado para el diagrama final.
AGENTS.md         Instrucciones para agentes que trabajen en este repo.
```

## Estado Actual

Este repositorio esta en fase de discovery y estrategia. No hay implementacion
de producto, base de datos, MCP server ni dashboard todavia. La prioridad es
entender bien el problema, comparar opciones y documentar decisiones antes de
dibujar la arquitectura final.

## Como Leer la Propuesta

1. Empezar por [docs/discovery/problem-framing.md](docs/discovery/problem-framing.md).
2. Revisar los retos de usuarios en [docs/discovery/user-challenges.md](docs/discovery/user-challenges.md).
3. Comparar alternativas en [docs/strategy/solution-options.md](docs/strategy/solution-options.md).
4. Leer la decision propuesta en [docs/adr/0001-adopt-mcp-first-data-access-strategy.md](docs/adr/0001-adopt-mcp-first-data-access-strategy.md).
5. Dejar el diagrama de arquitectura para el final, en [docs/architecture/README.md](docs/architecture/README.md).
