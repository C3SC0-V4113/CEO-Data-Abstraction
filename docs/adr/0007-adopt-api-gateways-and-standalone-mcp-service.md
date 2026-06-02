# ADR-0007: Adoptar API Gateways y Servicio MCP Independiente

## Status

Proposed

## Date

2026-06-02

## Context

La arquitectura vigente (ADR-0002, `docs/architecture/proposal.md`,
`docs/strategy/mcp-first-access.md`) tiene dos caracteristicas que el negocio pidio
cambiar:

1. **Sin control de trafico gobernado.** El frontend en Cloudflare llama directo al
   backend Fastify y los clientes MCP externos pegan directo a `POST /mcp`. No hay una
   capa que aplique rate limiting, throttling, cuotas, WAF, reglas de IP ni routing
   centralizado. Un cliente abusivo o un pico de trafico golpea directo al backend y al
   pipeline LLM (que ademas tiene costo por token).
2. **MCP acoplado al backend.** El servidor MCP vive dentro del runtime de Fastify.
   Esto mezcla el canal externo (programatico, autenticado por API key) con el backend
   web, dificulta escalar o limitar cada canal por separado y amplia la superficie del
   backend principal.

El stack es Cloudflare (frontend) + Railway (backend Fastify + PostgreSQL). No hay
data warehouse ni multiproveedor cloud. El paradigma chatbot-first (ADR-0005) y la capa
semantica (ADR-0006) ya estan decididos; este cambio es de topologia de servicios y
borde, no del pipeline de datos.

## Decision Drivers

- Controlar el trafico de la aplicacion (rate limiting, throttling, cuotas, WAF) antes
  de que llegue al backend y al pipeline LLM con costo por token.
- Aislar el canal MCP (programatico) del backend web (interactivo) con politicas propias.
- Mantener una sola Capa Semantica + SQL Safety Layer + acceso read-only + auditoria
  (no duplicar gobernanza ni credenciales de DB).
- Encajar en el stack actual (Cloudflare + Railway) con minimo costo operativo.
- No acoplar seguridad a una sola barrera.

## Considered Options

### API Gateway

#### Opcion A: Cloudflare como gateway de borde (elegida)

- Pros: el frontend ya esta en Cloudflare; API Shield + WAF + Rate Limiting + reglas sin
  operar un componente extra; bajo costo/ops para el MVP.
- Cons: politicas atadas al ecosistema Cloudflare.

#### Opcion B: Gateway dedicado self-host (Kong/KrakenD/Tyk)

- Pros: portable, control fino de politicas.
- Cons: agrega un servicio que desplegar y operar.

#### Opcion C: Cloud-managed (AWS/GCP/Azure APIM)

- Pros: gestionado y potente.
- Cons: introduce otro proveedor cloud fuera de Cloudflare/Railway.

### Topologia de gateway

- **Uno por servicio (elegida):** un Web API Gateway frente al backend Fastify y un MCP
  API Gateway frente al servicio MCP. Permite cuotas y politicas independientes por canal.
- Uno compartido por host/path: mas simple pero mezcla politicas de ambos canales.

### Reuso del pipeline por el servicio MCP

- **Adapter delgado -> core API interno (elegida):** el servicio MCP solo maneja protocolo
  MCP + auth (`MCP_API_KEY`) + scope por tool y llama a una API interna "core" del backend
  Fastify. Una sola Safety Layer, un solo acceso read-only y una sola auditoria.
- Paquete compartido + DB directa: sin salto extra pero duplica runtime y conexiones a DB.
- MCP totalmente standalone: maximo aislamiento, maxima duplicacion y riesgo de divergencia
  en seguridad.

## Decision

- Adoptar **Cloudflare como API Gateway de borde**, con **un gateway por servicio**: un
  **Web API Gateway** frente al backend Fastify y un **MCP API Gateway** frente al servicio
  MCP. Ambos aplican TLS, rate limiting, throttling, cuotas, WAF/IP rules, limite de tamano,
  routing y chequeos coarse (presencia de JWT/API key), mas observabilidad.
- **Separar el MCP en un servicio independiente** (desplegable en Railway, junto al backend),
  implementado como **adapter delgado**: maneja el protocolo MCP, valida `MCP_API_KEY` y el
  scope por tool, y mapea cada tool a una **API interna "core"** del backend Fastify
  (`/internal/core/*`). El servicio MCP **no** accede a la base de datos ni porta credenciales
  de DB/LLM.
- La **API interna core** del backend expone el pipeline (Orchestrator + Capa Semantica + SQL
  Safety Layer + acceso read-only + auditoria) a llamadores internos de confianza con **auth
  service-to-service** (`CORE_SERVICE_TOKEN` y/o red privada de Railway). No es publica; no pasa
  por el Web API Gateway.
- Reparto de responsabilidades: **gateways = control de trafico + auth coarse**; **servicios =
  authz fina** (backend valida firma JWT + rol; servicio MCP valida `MCP_API_KEY` + scope).

## Rationale

Cloudflare ya es parte del stack, asi que usarlo como gateway agrega control de trafico sin un
componente nuevo que operar. Un gateway por servicio permite afinar el canal MCP (cuotas por API
key, limites estrictos para clientes programaticos) distinto del canal web (rafagas
interactivas del CEO). El adapter delgado mantiene intactos los principios del proyecto:
una sola Safety Layer, un solo rol read-only y una sola auditoria; el servicio MCP nunca toca
la DB, reforzando "separar tokens MCP de credenciales de DB". El salto de red interno extra es
un costo aceptable frente a la duplicacion del pipeline y el riesgo de divergencia de seguridad.

## Consequences

### Positive

- Control de trafico (rate limit/throttling/cuotas/WAF) antes del backend y del costo LLM.
- Canal MCP aislado del backend web, con politicas y escalado propios.
- Gobernanza centralizada: una Safety Layer, un read-only, una auditoria.
- El servicio MCP no porta credenciales de DB; menor superficie de riesgo.

### Negative

- Mas piezas desplegables (dos gateways + servicio MCP) y mas configuracion.
- Un salto de red interno adicional (servicio MCP -> core API) con algo de latencia.
- Hay que gestionar el secreto service-to-service y la red privada.

### Risks and Mitigations

- Riesgo: la core API interna queda expuesta publicamente por error.
  Mitigacion: no enrutarla por el Web API Gateway; red privada de Railway + `CORE_SERVICE_TOKEN`;
  rechazar llamadas sin token de servicio valido.
- Riesgo: rate limits mal calibrados bloquean al CEO o no frenan abuso.
  Mitigacion: politicas separadas por gateway; iniciar conservador y ajustar con metricas.
- Riesgo: el servicio MCP duplica logica con el tiempo.
  Mitigacion: el adapter solo traduce protocolo; toda la logica de datos vive en la core API.
- Riesgo: el salto extra agrega latencia perceptible.
  Mitigacion: co-ubicar servicio MCP y backend en la misma region/red privada de Railway.

## Implementation Notes

- Endpoints: `POST /mcp` vive ahora en el **servicio MCP** (su propio host detras del MCP
  Gateway), no en Fastify. El backend expone `POST /internal/core/ask` (y afines) para uso
  interno del servicio MCP.
- Variables de entorno nuevas: `CORE_INTERNAL_URL` y `CORE_SERVICE_TOKEN` (servicio MCP ->
  backend). `MCP_API_KEY` ahora se configura en el servicio MCP. Config de rate limit/cuotas
  por gateway en Cloudflare.
- El servicio MCP no recibe `DATABASE_URL_*` ni claves del proveedor LLM.
- Despliegue sugerido MVP: servicio MCP como segundo servicio en Railway, misma region que el
  backend; ambos detras de gateways Cloudflare.

## Related Decisions

- ADR-0002: Next.js SSR, Fastify, Prisma y MCP remoto. Este ADR **refina** su decision de
  ubicar el MCP dentro de Fastify (ahora servicio independiente) y agrega el borde de gateways;
  el resto de ADR-0002 sigue vigente.
- ADR-0006: Capa semantica de metricas + estrategia de modelos. La core API expone ese pipeline;
  el servicio MCP lo consume sin duplicarlo.

## References

- `docs/architecture/proposal.md`
- `docs/strategy/mcp-first-access.md`
- `docs/diagrams/mermaid/architecture.mmd`, `docs/diagrams/mermaid/mcp-flow.mmd`
- Cloudflare API Shield / Rate Limiting / WAF: https://developers.cloudflare.com/api-shield/
