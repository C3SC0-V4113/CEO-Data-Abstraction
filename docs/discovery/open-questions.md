# MVP Decisions and Remaining Questions

## Decisiones Cerradas para MVP

### Negocio

- Usuario principal: CEO de una empresa desarrolladora de software.
- Roles: solo `CEO` en MVP.
- No habra registro publico ni multiusuario.
- El objetivo es que el CEO entienda salud ejecutiva de la empresa desde una
  experiencia de chatbot guiado.

### Metricas Oficiales MVP

- Revenue: MRR, ARR, crecimiento MRR, expansion revenue.
- Clientes: churn rate y clientes en riesgo.
- Comercial: pipeline por etapa y forecast de cierre.
- Delivery: proyectos en riesgo y margen por proyecto.
- Soporte: tickets criticos y SLA.
- Finanzas: burn rate, runway y costos por area.

### Metricas Exploratorias

Cualquier pregunta del chat que requiera consultar datos debe pasar por SQL
Safety Layer, allowlists, autorizacion por rol y permisos read-only antes de
ejecutar. Los reportes generados bajo demanda se consideran artefactos de chat,
no vistas separadas.

### Decisiones Esperadas del CEO

- Priorizar proyectos en riesgo.
- Revisar clientes con churn risk alto.
- Evaluar si runway y burn rate requieren accion.
- Revisar pipeline y forecast comercial.
- Identificar problemas de soporte que impactan cuentas clave.
- Pedir explicaciones de seguimiento sobre un artefacto generado en el chat.

### Datos

- Fuente MVP: PostgreSQL en Railway con datos ficticios generados por Prisma
  seed.
- Historico: 12 a 18 meses ficticios.
- Calidad de datos: suficiente para demo/MVP.
- El seed debe incluir anomalies intencionales para probar alertas, explicacion
  de cambios y preguntas contextuales.
- Moneda base: USD para MVP.
- Descuentos, credit notes y cancelaciones se simulan solo si ayudan a explicar
  churn, expansion revenue o variacion de MRR.

### IA y Seguridad

- Login obligatorio para la web.
- Usuario CEO creado por seed/setup.
- Sesion web con JWT.
- Rol `CEO` incluido en claims y validado por backend.
- Credenciales reales no se documentan en texto plano.
- MCP usa `MCP_API_KEY` y queda como canal externo para clientes compatibles.
- Web no consume MCP directamente; consume APIs Fastify.
- Auditoria: prompts, report context, SQL generado, SQL validado, cliente,
  usuario, rows, warnings y `trace_id`.
- Reportes automaticos no forman parte del MVP. Si se agregan despues, deben
  tratarse como nueva decision.

### Capa Semantica y Modelos (ADR-0006)

- Se adopta una capa de metricas headless propia. El camino por defecto es
  NL -> `MetricQuery` -> SQL determinista sobre views `ceo_*`, no SQL libre.
- Text-to-SQL libre queda como fallback auditado para preguntas fuera del catalogo.
- El schema se entrega al LLM como catalogo de metricas compacto y cacheable, no como
  DDL crudo; RAG sobre definiciones solo si el catalogo crece.
- La capa de modelos es agnostica de proveedor con routing de dos niveles
  (ligero + planificador). Default razonado: Claude Sonnet/Haiku, intercambiable por
  configuracion. La decision por costo se apoya en `docs/cost/llm-cost-calculator.xlsx`.

### API Gateways y Servicio MCP (ADR-0007)

- El trafico entra por dos API Gateways Cloudflare (uno por servicio): web y MCP, con
  rate limiting, throttling, cuotas, WAF y routing.
- El MCP deja de vivir dentro de Fastify y pasa a ser un servicio independiente (adapter
  delgado) que llama a la Core Internal API del backend con auth service-to-service.
- El servicio MCP no porta credenciales de DB ni LLM; una sola Safety Layer + read-only +
  auditoria en el backend core.

### Experiencia

- Login y chatbot son las unicas interfaces principales del MVP.
- Los reportes se generan dentro de la vista de chat segun la pregunta.
- El chatbot debe sugerir preguntas y acciones rapidas para evitar un chat
  vacio.
- Consultas sugeridas por defecto:
  - "Que cambio mas desde el ultimo periodo?"
  - "Que proyectos requieren atencion?"
  - "Que clientes tienen mayor riesgo?"
  - "Que explica la variacion de MRR?"
  - "Que tickets criticos afectan cuentas clave?"
- Medicion de exito: el CEO puede obtener respuestas con narrativa, tablas y
  graficas dentro del chat sin conocer SQL ni estructura de tablas.

## Preguntas Futuras

- Incluir CFO, COO o lideres de area.
- Conectar datos reales desde sistemas transaccionales, warehouse o archivos.
- Definir multi-moneda y reglas contables reales.
- Activar reportes enviados por correo o Slack.
- Evaluar tiempo real o near-real-time para soporte/proyectos.
- Confirmar precios vigentes por proveedor para cerrar el presupuesto de la
  calculadora de costos.
- Definir el umbral de cobertura del catalogo de metricas (que % de preguntas debe
  resolver el camino semantico antes de aceptar el fallback).
- Decidir cuando activar RAG/embeddings sobre el catalogo (a partir de que tamano).
- Evaluar si conviene migrar la capa semantica a dbt Semantic Layer al escalar a
  multiples roles y fuentes.
- Calibrar los limites de rate limiting / cuotas por canal (web vs MCP).
- Decidir red privada de Railway vs solo `CORE_SERVICE_TOKEN` para proteger la Core
  Internal API.
- Definir el despliegue concreto del servicio MCP (region, escalado) y su observabilidad.
