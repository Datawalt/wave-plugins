<p align="center">
  <img src=".claude-plugin/icon.png" alt="Wave by Datawalt" width="120" height="120" />
</p>

<h1 align="center">Wave</h1>
<p align="center"><strong>Habla con tus datos. Sin SQL, sin esperar al equipo de BI.</strong></p>
<p align="center">Por <a href="https://datawalt.cl">Datawalt</a> · BI conversacional sobre tu modelo semántico</p>

---

Wave es la capa de **inteligencia de negocio conversacional** de Datawalt. Este plugin trae Wave a Claude Code: pregúntale a tus datos en lenguaje natural y recibe respuestas en segundos, con números reales de tu empresa.

## Por qué Wave

- **Cero SQL.** Tus preguntas en castellano. Wave traduce a tu modelo semántico y te devuelve la respuesta.
- **Tus métricas, no estimaciones.** Las definiciones las cura el equipo Datawalt + tu negocio. Un solo número, una sola verdad.
- **Comparaciones y tendencias en un click.** Q1 vs Q1, top-N, outliers, year-over-year — sin armar tablas.
- **Recomendaciones gerenciales.** No solo "qué pasó", también "dónde enfocar al equipo este mes".
- **Solo lectura, listo en segundos.** Cero riesgo de romper nada. El modelo y los dashboards los gestiona el equipo Datawalt.

## Qué puedes preguntar

- "¿Cuánto vendí en marzo?"
- "Compara Q1 2026 vs Q1 2025."
- "¿Qué cliente compró más este trimestre?"
- "Top 10 productos por margen."
- "¿Dónde debería enfocar al equipo comercial este mes?"
- "¿Qué pasó con la tasa de conversión la semana pasada?"

## Skills incluidas

| Skill | Para qué |
|---|---|
| **`wave-ask`** | Punto de entrada para conversación con tus datos. Tres modos según la pregunta: cerrado (un número puntual), comparativo (tendencias, top-N, variaciones, outliers) y prescriptivo (recomendaciones accionables priorizadas). |
| **`wave-explore-data`** | Descubrir qué métricas, modelos y dashboards están disponibles. |

> Post-ADR-0017 PR4 (mayo 2026): `wave-insights` y `wave-recommendation` fueron absorbidas como modos del system prompt de `wave-ask`. El plugin pasa de 4 a 2 skills.

## Autenticación

El plugin se conecta al servidor MCP de Wave en `https://mcp.datawalt.app` y autentica con **OAuth 2.1 + PKCE** contra Cognito. **No necesitas configurar tokens ni variables de entorno**: la primera vez que uses el plugin, Claude Code abre tu navegador para que inicies sesión con tu cuenta Datawalt; el access token se guarda de forma segura y se refresca automáticamente.

## Instalación

```
/plugin install ./plugins/wave
```

Una vez instalado, valida:

- `/mcp` debe listar el conector **Wave**.
- `/skill` debe listar `wave-ask` y `wave-explore-data` (post-ADR-0017 PR4).

## Primera consulta

Abre Claude Code y prueba alguna de estas:

- "¿Con qué cuenta estoy conectado?"
- "¿Qué puedo consultar?"
- "¿Cuánto vendí en marzo?"
- "Compara Q1 2026 vs Q1 2025."
- "¿Dónde debería enfocar al equipo comercial este mes?"

Si tu cuenta tiene acceso a varias empresas, te pedirán especificar el subdominio (ej: `miempresa`) en la primera consulta.

## Soporte

Contacta al administrador de Wave en tu empresa o al equipo Datawalt en [datawalt.cl](https://datawalt.cl).

---

<p align="center"><sub>Wave by <a href="https://datawalt.cl">Datawalt</a> · Datos que hablan tu idioma.</sub></p>
