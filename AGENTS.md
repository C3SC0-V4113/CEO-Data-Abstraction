# AGENTS.md

Instrucciones para agentes y asistentes que trabajen en este repositorio.

## Idioma y Estilo

- Escribir la documentacion en espanol claro y tecnico.
- Mantener el tono pragmatico: discovery primero, decision despues,
  implementacion al final.
- Evitar vender la idea como producto terminado. Este repo esta explorando una
  estrategia.

## Prioridades

- Priorizar discovery, estrategia y ADRs antes de proponer codigo.
- Mantener las automatizaciones como hipotesis de brainstorming hasta que un ADR
  las acepte formalmente.
- No asumir que existe una base de datos productiva. Usar el dominio Ventas CRM
  solo como caso de referencia.
- No permitir SQL libre generado por un LLM como recomendacion por defecto.
  Preferir herramientas MCP allowlisted, metricas gobernadas y capa semantica.

## Documentacion

- Las decisiones relevantes deben registrarse en `docs/adr/`.
- Las dudas, supuestos y preguntas abiertas deben ir en `docs/discovery/`.
- Los terminos ambiguos deben agregarse al glosario en `docs/glossary/README.md`.
- Las opciones de solucion deben compararse en `docs/strategy/`.
- El diagrama de arquitectura se agregara despues de estabilizar la estrategia.

## ADRs

- Usar el formato de `docs/adr/template.md`.
- Usar estados: `Proposed`, `Accepted`, `Rejected`, `Deprecated` o
  `Superseded`.
- No cambiar una decision aceptada para reescribir historia. Crear un nuevo ADR
  que la superseda.

## Contexto del Autor

El propietario del ejercicio tiene background como ingeniero en ciencias de la
computacion, conocimiento general de bases de datos, redes y matematicas,
especialidad frontend con React y experiencia con aplicaciones de IA como
chatbots, transcriptores y TTS de OpenAI. Las propuestas pueden asumir ese nivel
tecnico, pero deben seguir siendo presentables a negocio.
