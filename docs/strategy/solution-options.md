# Solution Options

## Opcion Adoptada: Web Chat-First + MCP sobre Fastify + Prisma

La solucion propuesta combina una web app ejecutiva en Next.js SSR + shadcn/ui
con un servidor MCP remoto. Ambos frentes reutilizan el **mismo pipeline Text-to-SQL
seguro** (Fastify + TypeScript + Prisma): la web lo consume por sus APIs HTTP y el MCP a
traves de un **servicio independiente** que llama a la Core Internal API del backend. El
trafico entra por **API Gateways (Cloudflare)**, uno por servicio (ver ADR-0007).

La web deja de ser dashboard/report-first. El MVP tendra login y una vista de
chatbot donde el CEO solicita analisis, recibe narrativa ejecutiva y ve reportes
generados bajo demanda dentro de la propia conversacion.

### Cuando Tiene Sentido

- El entregable exige web app y MCP.
- El usuario principal es un CEO que necesita respuestas ejecutivas sin conocer
  SQL ni estructura de tablas.
- Se quiere una interfaz minima: login y chatbot.
- Se requiere seguridad estricta, JWT, roles, auditoria y SQL read-only.
- Se quiere mantener TypeScript end-to-end para web, API, MCP y orquestacion.

### Ventajas

- Unifica logica de negocio para web y MCP.
- Next.js SSR permite una experiencia web moderna sin construir multiples
  pantallas analiticas.
- Fastify encaja con TypeScript, APIs de alto rendimiento, validacion y MCP.
- Prisma aporta modelos tipados, migraciones y conexion limpia a PostgreSQL.
- PostgreSQL permite roles read-only reales.
- El chatbot puede generar tablas, graficas y KPIs segun la pregunta.

### Riesgos

- Un chat vacio puede depender demasiado de buenas preguntas sugeridas.
- La continuidad conversacional debe manejar filtros y periodos con rigor.
- Prisma en Cloudflare Workers requiere validacion edge y posiblemente Prisma
  Accelerate si el backend se mueve fuera de Railway.
- Requiere una capa seria de validacion SQL.

## Alternativa: Dashboard Report-First

La aplicacion abre en un dashboard ejecutivo con reportes predefinidos y chat
contextual secundario.

### Ventajas

- Reduce la necesidad de que el CEO formule preguntas desde cero.
- Encaja bien con vistas SSR separadas y snapshots precomputados.
- Facilita revisar KPIs frecuentes.

### Riesgos

- Ya no representa el cambio de paradigma actual.
- Aumenta alcance de UI con rutas, widgets, filtros e historico.
- Subordina el chatbot cuando la experiencia deseada ahora es conversacional.

## Alternativa: Next.js Fullstack Unico

Toda la aplicacion vive en Next.js: UI, APIs, autenticacion y llamadas al LLM.

### Ventajas

- Menos runtimes.
- TypeScript end-to-end.
- Despliegue SSR claro en Cloudflare Workers.

### Riesgos

- El servidor MCP y la validacion SQL pueden quedar mas acoplados al frontend.
- No aprovecha Fastify como backend especializado.

## Alternativa: Fastify + Frontend Estatico

Frontend estatico y backend Fastify central.

### Ventajas

- Simple para hosting.
- Backend fuerte para data e IA.

### Riesgos

- No cumple la decision de usar Next.js SSR.
- Menor capacidad de experiencia dinamica desde el servidor.

## Alternativa: Cloudflare D1 como Base Principal

Usar D1 para mantener todo dentro de Cloudflare.

### Ventajas

- Muy alineado con Cloudflare.
- Practico para prototipo gratuito.
- Menos servicios externos.

### Riesgos

- No cubre tan bien el requisito de permisos read-only `SELECT` a nivel de motor.
- PostgreSQL es mas adecuado para roles, views, analitica y portabilidad.

## Despliegue Backend

### Railway

Railway es la opcion recomendada para el MVP del backend Fastify + Prisma y
PostgreSQL:

- Node.js tradicional.
- Prisma Client funciona de forma directa.
- Buen encaje con Railway Postgres o Postgres compatible.
- Menos friccion para MCP SDK, build y dependencias.
- Permite mantener backend y base de datos en el mismo proveedor durante MVP.

### Cloudflare Workers

Cloudflare Workers sigue siendo opcion para backend si la prueba tecnica valida:

- Fastify bajo Node.js compatibility.
- Prisma edge client o Prisma Accelerate.
- MCP SDK compatible con el runtime.
- Conexion estable a PostgreSQL serverless.

## Complementos Futuros

Estas ideas siguen siendo validas, pero no son el nucleo del requerimiento:

- Dashboards persistentes.
- Historico formal de reportes.
- Reportes automaticos.
- Alertas inteligentes.
- RAG para definiciones de negocio, solo si una decision futura lo vuelve necesario.
- Exportacion y programacion de reportes.

## Lectura Recomendada

La arquitectura base debe resolver primero login con JWT, chatbot ejecutivo,
backend Fastify + Prisma, MCP remoto, Text-to-SQL seguro, PostgreSQL read-only y
auditoria. Las automatizaciones, dashboards persistentes y reportes proactivos
deben tratarse como extensiones posteriores.
