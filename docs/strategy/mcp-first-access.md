# MCP Remote Access

## Proposito

MCP ya no es solo una opcion de arquitectura: es un entregable obligatorio. El
sistema debe exponer un servidor MCP remoto para que Claude Desktop,
Cursor/Codex u otros clientes compatibles puedan consultar los mismos datos que
la web app.

## Relacion con Fastify

El servidor MCP vivira dentro del backend Fastify o como modulo montado en el
mismo runtime. Las tools MCP no duplican logica: llaman los mismos servicios
internos que usa `POST /api/chat/messages`.

MCP no es la API del frontend. La web SSR consume endpoints HTTP propios de
Fastify, como `/api/auth/*`, `/api/chat/*` y `/api/schema/*`. MCP queda como
canal externo para Claude Desktop, Cursor/Codex u otros clientes compatibles.

## Endpoint

El endpoint objetivo sera:

```text
POST https://<host>/mcp
Authorization: Bearer <token>
```

El transporte debe ser HTTP remoto compatible con MCP. El servidor debe validar
token/API key antes de exponer schema, tools o cualquier metadata.

El token se configurara como secreto `MCP_API_KEY`. El frontend no debe recibir
ni usar este secreto.

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

- Requerir bearer token en cada request.
- No exponer schema antes de autenticar.
- Separar tokens MCP de credenciales de base de datos y proveedor LLM.
- Separar MCP auth de web auth.
- No exponer `MCP_API_KEY` al browser.
- Aplicar scope por tool.
- Auditar cliente, usuario, pregunta, SQL generado, SQL validado y resultado.
- Ejecutar contra PostgreSQL con usuario read-only.

## Flujo End-to-End

1. Cliente MCP externo envia pregunta con token.
2. Fastify valida token y scope.
3. MCP tool llama al LLM Orchestrator interno.
4. El orchestrator genera SQL candidato.
5. SQL Safety Layer valida por AST y allowlists.
6. Database Layer ejecuta con usuario read-only.
7. Visualization Layer sugiere grafica.
8. MCP responde con datos, narrativa y `trace_id`.
