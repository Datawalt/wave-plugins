# wave

Plugin de Wave BI para usuarios finales. Te permite **preguntarle a tus datos en lenguaje natural** desde Claude Code: cuánto vendiste, cuántos clientes activos tienes, qué pasó con tus métricas, qué deberías priorizar.

## Qué puedes hacer

- **Preguntar por tus números**: "cuánto vendí en marzo", "qué cliente compró más", "ventas por sucursal este trimestre".
- **Explorar qué hay disponible**: descubrir modelos, métricas y dashboards que tu empresa ya tiene configurados.
- **Análisis comparativos**: comparar periodos, ver tendencias, top-N, detectar outliers.
- **Recomendaciones gerenciales**: dónde enfocar al equipo, qué oportunidades surgen de los datos.

Estricto **read-only**. No puedes modificar el modelo ni los dashboards desde aquí — eso lo hace el equipo Datawalt.

## Skills incluidas

| Skill | Para qué |
|---|---|
| `wave-ask` | Preguntas directas de negocio en lenguaje natural. |
| `wave-explore-data` | Descubrir qué métricas, modelos y dashboards están disponibles. |
| `wave-insights` | Análisis comparativos: tendencias, top-N, variaciones, outliers. |
| `wave-recommendation` | Recomendaciones accionables basadas en tus datos. |

## Autenticación

El plugin se conecta al servidor MCP de Wave en `https://mcp.datawalt.app` y autentica con OAuth 2.1 + PKCE contra Cognito. **No necesitas configurar tokens ni variables de entorno**: la primera vez que uses el plugin, Claude Code abre tu navegador para que inicies sesión con tu cuenta de Datawalt; el flow guarda el access token de forma segura y lo refresca automáticamente.

## Instalación

```
/plugin install ./plugins/wave
```

Una vez instalado, valida:

- `/mcp` debe listar el servidor `wave`.
- `/skill` debe listar `wave-ask`, `wave-explore-data`, `wave-insights`, `wave-recommendation`.

## Primera consulta

Abre Claude Code y prueba alguna de estas:

- "¿Con qué cuenta estoy conectado?"
- "¿Qué puedo consultar?"
- "¿Cuánto vendí en marzo?"
- "Compara Q1 2026 vs Q1 2025."
- "¿Dónde debería enfocar al equipo comercial este mes?"

Si tu cuenta tiene acceso a varias empresas, te pedirán especificar el subdominio (ej: `miempresa`) en la primera consulta.

## Soporte

Contacta al administrador de Wave en tu empresa o al equipo Datawalt.
