# MVP Decisions and Remaining Questions

## Decisiones Cerradas para MVP

### Negocio

- Usuario principal: CEO de una empresa desarrolladora de software.
- Roles: solo `CEO` en MVP.
- No habra registro publico ni multiusuario.
- El objetivo es que el CEO entienda salud ejecutiva de la empresa sin escribir
  prompts.

### Metricas Oficiales MVP

- Revenue: MRR, ARR, crecimiento MRR, expansion revenue.
- Clientes: churn rate y clientes en riesgo.
- Comercial: pipeline por etapa y forecast de cierre.
- Delivery: proyectos en riesgo y margen por proyecto.
- Soporte: tickets criticos y SLA.
- Finanzas: burn rate, runway y costos por area.

### Metricas Exploratorias

Cualquier pregunta ad hoc del chat que no este cubierta por widgets o reportes
predefinidos se considera exploratoria. Debe pasar por SQL Safety Layer,
allowlists y permisos read-only antes de ejecutar.

### Decisiones Esperadas del CEO

- Priorizar proyectos en riesgo.
- Revisar clientes con churn risk alto.
- Evaluar si runway y burn rate requieren accion.
- Revisar pipeline y forecast comercial.
- Identificar problemas de soporte que impactan cuentas clave.
- Preguntar por explicaciones especificas desde un reporte.

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
- Credenciales reales no se documentan en texto plano.
- MCP usa `MCP_API_KEY` y queda como canal externo para clientes compatibles.
- Web no consume MCP directamente; consume APIs Fastify.
- Auditoria: prompts, report context, SQL generado, SQL validado, cliente,
  usuario, rows, warnings y `trace_id`.
- Reportes automaticos no requieren aprobacion humana en MVP porque no se envian
  externamente; solo se consultan desde el dashboard.

### Experiencia

- Dashboard/reportes son la experiencia principal.
- Chat global y chat contextual son secundarios.
- Cada grafico o reporte relevante debe tener accion `Preguntar`.
- Consultas sugeridas por defecto:
  - "Que cambio mas desde el ultimo periodo?"
  - "Que proyectos requieren atencion?"
  - "Que clientes tienen mayor riesgo?"
  - "Que explica la variacion de MRR?"
  - "Que tickets criticos afectan cuentas clave?"
- Medicion de exito: el CEO puede entender overview, revenue, clientes,
  pipeline, proyectos, soporte y finanzas sin abrir el chat.

## Preguntas Futuras

- Incluir CFO, COO o lideres de area.
- Conectar datos reales desde sistemas transaccionales, warehouse o archivos.
- Definir multi-moneda y reglas contables reales.
- Activar reportes enviados por correo o Slack.
- Evaluar tiempo real o near-real-time para soporte/proyectos.
