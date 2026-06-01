# Solution Options

## Opcion Adoptada: Web SSR + MCP sobre Fastify + Prisma

La solucion propuesta combina una web app ejecutiva en Next.js SSR + shadcn/ui
con un servidor MCP remoto. Ambos frentes consumen el mismo backend Fastify +
TypeScript, Prisma ORM y el mismo pipeline Text-to-SQL seguro.

### Cuando Tiene Sentido

- El entregable exige web app y MCP.
- El usuario principal es un CEO que necesita graficas y respuestas ejecutivas.
- Se requiere seguridad estricta y auditoria.
- Se quiere mantener TypeScript end-to-end para web, API, MCP y orquestacion.

### Ventajas

- Unifica logica de negocio para web y MCP.
- Next.js SSR permite una experiencia web moderna.
- Fastify encaja con TypeScript, APIs de alto rendimiento, validacion y MCP.
- Prisma aporta modelos tipados, migraciones y conexion limpia a PostgreSQL.
- PostgreSQL permite roles read-only reales.

### Riesgos

- Hay mas piezas de despliegue que en una app monolitica.
- Prisma en Cloudflare Workers requiere validacion edge y posiblemente Prisma
  Accelerate.
- Requiere una capa seria de validacion SQL.

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

- No cumple la decision de usar Next.js fullstack con SSR.
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

- Preguntas sugeridas para CEO.
- Catalogo de metricas ejecutivas.
- Reportes automaticos.
- Alertas inteligentes.
- RAG para definiciones de negocio.

## Lectura Recomendada

La arquitectura base debe resolver primero web SSR, backend Fastify + Prisma,
MCP remoto, Text-to-SQL seguro, PostgreSQL read-only y auditoria. Las
automatizaciones y reportes proactivos deben tratarse como extensiones
posteriores.
