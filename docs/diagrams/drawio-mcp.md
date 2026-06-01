# Draw.io MCP en Codex

El MCP de draw.io se configura en Codex, no en Claude Desktop.

## Configuracion

Agregar al archivo global de Codex:

```toml
[mcp_servers.drawio]
command = "npx"
args = ["-y", "@drawio/mcp"]
startup_timeout_sec = 120
```

Ruta local:

```text
C:\Users\frank\.codex\config.toml
```

## Tools Esperadas

El MCP oficial de draw.io expone tools para abrir contenido en draw.io:

- `open_drawio_xml`
- `open_drawio_mermaid`
- `open_drawio_csv`

## Flujo

1. Crear o editar Mermaid/XML en este repo.
2. Abrir el contenido con el MCP de draw.io.
3. Ajustar visualmente en draw.io.
4. Guardar el resultado como `.drawio`.

## Notas

- El primer arranque puede ejecutar `npx -y @drawio/mcp` y requerir red.
- Tras modificar `config.toml`, Codex puede necesitar reiniciar o recargar MCPs.
- Los archivos Mermaid y XML se mantienen en el repo aunque se usen `.drawio`
  para presentacion.
