# Glosario

## Agente

Sistema basado en IA que puede razonar sobre una tarea, usar herramientas,
consultar datos y producir una respuesta o accion.

## Alerta Inteligente

Notificacion generada cuando una metrica cruza un umbral, cambia de forma
anomala o requiere atencion segun reglas y contexto.

## Capa Semantica

Definicion gobernada de metricas, dimensiones, filtros y relaciones. Evita que
cada usuario o modelo interprete los datos de forma distinta.

## Copiloto

Experiencia asistida por IA que acompana al usuario dentro de una interfaz,
normalmente combinando acciones guiadas, explicaciones y conversacion.

## Forecast

Pronostico de una metrica futura, por ejemplo ventas esperadas del proximo mes.
Debe indicar horizonte, supuestos y nivel de confianza.

## Job

Proceso automatizado que corre bajo una frecuencia o condicion. En este repo los
jobs son una hipotesis para reportes, snapshots, alertas y analisis pesados.

## MCP

Model Context Protocol. Protocolo para exponer herramientas y contexto a modelos
o agentes como Codex y Claude.

## Metrica

Valor cuantificable de negocio, como ventas totales, margen, cumplimiento de
meta o ticket promedio.

## Pregunta Sugerida

Pregunta que el sistema propone al usuario segun su contexto, rol, datos
recientes o anomalias detectadas.

## RAG

Retrieval-Augmented Generation. Patron donde el modelo recupera informacion
relevante desde documentos o bases de conocimiento antes de responder.

## Snapshot

Resultado precomputado de una metrica en un momento o periodo. Sirve para
consultas frecuentes, comparativas o reportes pesados.

## SQL Libre

SQL generado dinamicamente por un LLM sin restricciones suficientes. Se considera
riesgoso para este caso por seguridad, consistencia y auditabilidad.
