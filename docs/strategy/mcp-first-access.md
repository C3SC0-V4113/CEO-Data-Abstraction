# MCP Remote Access

## Proposito

MCP ya no es solo una opcion de arquitectura: es un entregable obligatorio. El
sistema debe exponer un servidor MCP remoto para que Claude Desktop,
Cursor/Codex u otros clientes compatibles puedan consultar los mismos datos que
la web app.

## Relacion con Fastify

El servidor MCP **ya no vive dentro de Fastify**: es un **servicio desplegable
independiente** (ver ADR-0007). Funciona como adapter delgado: maneja el protocolo MCP,
valida `MCP_API_KEY` y el scope por tool, y mapea cada tool a la **API interna core** del
backend Fastify (`/internal/core/*`). Asi no duplica logica ni accede a la base de datos:
toda la logica de datos (Orchestrator + Capa Semantica + SQL Safety Layer + read-only +
auditoria) vive en la core API.

MCP no es la API del frontend. La web SSR consume endpoints HTTP propios de Fastify, como
`/api/auth/*`, `/api/chat/*` y `/api/schema/*`, a traves del **Web API Gateway**. El
servicio MCP queda como canal externo para Claude Desktop, Cursor/Codex u otros clientes
compatibles, a traves del **MCP API Gateway**.

## Endpoint

El endpoint objetivo (ahora en el host del servicio MCP, detras del MCP API Gateway) sera:

```text
POST https://<host-mcp>/mcp
Authorization: Bearer <token>
```

El transporte debe ser HTTP remoto compatible con MCP. El servicio debe validar
token/API key antes de exponer schema, tools o cualquier metadata.

El token se configurara como secreto `MCP_API_KEY` en el servicio MCP. El frontend no
debe recibir ni usar este secreto. El servicio MCP llama a la core API sobre la **red
privada de Railway** (`*.railway.internal`) con `CORE_SERVICE_TOKEN` (auth
service-to-service); no recibe `DATABASE_URL_*` ni claves LLM.

## Tools MCP Candidatas

### describe_business_schema

Devuelve las views, metricas y columnas permitidas para el rol autenticado.

### ask_company_data

Recibe una pregunta ejecutiva en lenguaje natural y ejecuta el pipeline completo
Text-to-SQL: contexto, LLM, validacion, consulta read-only y respuesta.

### run_readonly_query

Ejecuta SQL solo si ya paso validacion estricta. Esta tool debe estar limitada a
usuarios o scopes tecnicos.

### suggest_executive_questions

Propone preguntas utiles para un CEO segun metricas disponibles y contexto.

### generate_chart_spec

Recibe datos tabulares o una consulta validada y propone una visualizacion:
`table`, `kpi`, `bar`, `line` o `area`.

## Respuesta Comun

Las tools deben devolver una estructura compatible con la web app:

- `answer`
- `sql`
- `data`
- `chart`
- `warnings`
- `metadata`
- `trace_id`

## Seguridad

- MCP API Gateway (Cloudflare) con rate limiting, cuotas por API key y WAF antes del servicio.
- Requerir bearer token en cada request.
- No exponer schema antes de autenticar.
- Separar tokens MCP de credenciales de base de datos y proveedor LLM.
- El servicio MCP no porta credenciales de DB ni LLM; llama a la core API con
  `CORE_SERVICE_TOKEN` (auth service-to-service).
- Separar MCP auth de web auth.
- No exponer `MCP_API_KEY` al browser.
- Aplicar scope por tool.
- Auditar cliente, usuario, pregunta, SQL generado, SQL validado y resultado (en el backend core).
- Ejecutar contra PostgreSQL con usuario read-only (solo desde el backend core).

## Flujo End-to-End

1. Cliente MCP externo envia pregunta con token.
2. La request pasa por el MCP API Gateway (rate limiting, cuotas, WAF).
3. El servicio MCP valida `MCP_API_KEY` y scope, y llama a la core API del backend
   (`/internal/core/*`) con `CORE_SERVICE_TOKEN`.
4. El LLM Orchestrator produce un `MetricQuery` (o fallback) y la Capa Semantica compila SQL.
5. SQL Safety Layer valida por AST y allowlists.
6. Database Layer ejecuta con usuario read-only.
7. El backend sugiere visualizacion y registra auditoria.
8. El backend responde al servicio MCP, que devuelve datos, narrativa y `trace_id` al cliente.
