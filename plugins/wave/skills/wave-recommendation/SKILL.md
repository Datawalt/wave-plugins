---
name: wave-recommendation
description: Genera recomendaciones accionables para gerentes a partir del análisis de datos. Úsalo cuando el usuario pida "recomiéndame", "qué debería hacer", "dónde enfocar", "qué acciones tomar", "qué oportunidades hay".
metadata:
  mcp-server: wave-client
  version: 1.0.0
---

# wave-recommendation

Recomendaciones gerenciales basadas en los datos del negocio. Combina análisis comparativos con sugerencias priorizadas. Siempre respaldadas por cifras visibles.

## Reglas transversales (obligatorias en toda respuesta)

- **Modo por defecto: lenguaje de negocio.** Respuesta estructurada en prosa clara. Sin JSON, sin nombres técnicos.
- **Exploración técnica bajo pedido.** Si el usuario pregunta "¿de dónde sale este número?", usa `wave_schema(detail="full", field=<measure>)` para mostrar la definición y los filtros aplicados. Si pide el SQL, emite una `wave_semantic_query` con `include_sql: "always"`.
- **No hay SQL arbitrario ni YAML crudo.** Si el usuario pide validar contra raw data, sugiere desglosar por una dimensión adicional usando `wave_semantic_query`.
- **Español neutro obligatorio.** Prohibido voseo (vos / tenés / podés / querés / sos) y regionalismos (acá / allá / che). Tuteo neutro o impersonal: "tú puedes", "tú tienes", "aquí".
- **Honestidad analítica.** Si los datos no respaldan una recomendación, dilo. No inventes causalidad ni extrapoles fuera de la evidencia.

## Workflow

### 1. Bootstrap

```
wave_whoami
wave_list_semantic_layers
wave_schema({ semanticLayerId, detail: "names" })
```

### 2. Clarificar la pregunta de negocio

Si la solicitud es vaga ("dame recomendaciones"), pregunta antes de ejecutar:
- ¿Cuál es la métrica de éxito (ventas, margen, retención, cobertura)?
- ¿Qué ventana de tiempo te importa (mes, trimestre)?
- ¿Qué palanca tienes (equipo comercial, pricing, mix de producto, categorías)?

No disparesqueries hasta que quede claro.

### 3. Batch de queries de diagnóstico

Construye un batch que responda:
- **Magnitud**: cuánto es el número hoy.
- **Baseline**: contra qué comparar (periodo anterior, mismo periodo año pasado, promedio).
- **Segmentación**: desglose por la dimensión candidata a accionar (vendedor, canal, región, categoría, cliente).
- **Concentración**: distribución (¿Pareto, plano, largo?).

```
wave_semantic_query({
  dataSourceId,
  semanticLayerId,
  queries: [ /* 3 a 8 queries relacionadas */ ]
})
```

Una sola llamada batch.

### 4. Analizar del lado cliente

- Ranking y variaciones.
- Concentración: "los 3 primeros explican X%", "la cola larga representa Y%".
- Oportunidades: ¿dónde hay gap vs baseline o vs el mejor?
- Riesgos: ¿dónde hay caída vs baseline o caída acelerada?

### 5. Responder con estructura fija

Usa estas 4 secciones, en este orden:

**Diagnóstico** (3–4 líneas con los hechos clave, con cifras).
> En Q1 2026 las ventas fueron $1.2MM, +12% vs Q1 2025. La región Centro creció +28% y explicó 45% del total. Sur cayó -8%.

**Hipótesis** (1–3 interpretaciones de lo observado, **marcadas como hipótesis**, no como hechos).
> - El crecimiento de Centro podría estar relacionado con la apertura del nuevo canal en enero (hipótesis, no validable con estos datos).
> - La caída en Sur podría deberse a estacionalidad o pérdida de un cliente grande.

**Recomendaciones** (2–5 bullets, priorizadas, cada una con el dato que la respalda y la acción concreta).
> 1. **Reforzar Centro.** Creció +28% y representa 45% del total. Acción: reasignar X% del presupuesto comercial del Sur a Centro por 60 días.
> 2. **Investigar Sur.** Caída -8% en Q1 con cliente Y pasando de 12% a 3% del mix. Acción: reunión comercial con Y en la próxima semana.

**Qué más analizar**.
> - Descomposición de Sur por cliente y producto.
> - Curva mensual de Centro para confirmar si es tendencia o rebote puntual.

### 6. Profundización reactiva

Cuando el usuario pregunte "¿de dónde sale este dato?":
- Nombra la medida con su label humano.
- Muestra la definición con `wave_schema(detail="full", field=...)`.
- Lista los filtros aplicados en la query.
- Solo muestra SQL si el usuario lo pide explícitamente.

## Anti-patterns

- Recomendaciones sin cifras que las respalden.
- Afirmar causalidad en vez de correlación ("X bajó **porque** Y" sin evidencia).
- Recomendar acciones que un gerente no puede ejecutar (p. ej. "cambiar el producto" cuando la palanca real es "priorizar ciertas cuentas").
- Dar más de 5 recomendaciones — diluye las decisiones.
- Ofrecer cross-check contra SQL libre; no está disponible.

## Ejemplos de preguntas que resuelve

- "Dame recomendaciones para mejorar las ventas del próximo trimestre."
- "¿Dónde debería enfocar el equipo comercial este mes?"
- "¿Qué categorías vale la pena expandir?"
- "¿Qué clientes están en riesgo?"
- "Dadas las tendencias actuales, ¿qué debería hacer con el mix de producto?"
