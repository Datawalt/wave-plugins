# Datawalt Wave plugins

Releases públicos de los plugins de Datawalt para Claude Code y Claude Cowork.

## Plugins distribuidos

| Plugin | Audiencia | Descripción |
|---|---|---|
| `wave` | Clientes finales | Pregunta a tus datos en lenguaje natural. Read-only. |

El plugin `wave` se conecta al servidor MCP en `mcp.datawalt.app/api/mcp/wave/handler` con OAuth/PKCE. La primera vez que se usa abre el navegador para iniciar sesión con Cognito; no requiere tokens manuales ni variables de entorno.

## Cómo se distribuye

### Claude Cowork (Teams / Enterprise)

El admin del cliente:

1. Descarga el `.zip` del último release público (sección Releases →).
2. En su panel admin de Cowork → Plugins → Private Marketplace → Upload plugin.
3. Asigna modo (`Required`, `Installed by default`, `Available for install` o `Not available`).

Los users de la organización ven el plugin en su Cowork y lo instalan a un click.

### Claude Code (cliente técnico)

```
/plugin marketplace add Datawalt/wave-plugins
/plugin install wave@datawalt-plugins
```
