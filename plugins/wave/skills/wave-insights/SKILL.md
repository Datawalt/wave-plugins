---
name: wave-insights
description: Genera análisis comparativos sobre los datos del negocio: comparación entre periodos, top-N, tendencias, variaciones y detección de outliers. Úsalo cuando el usuario pida "comparar", "ver tendencia", "top 10", "qué cambió", "mejor/peor cliente/producto/vendedor", "análisis de X".
metadata:
  mcp-server: wave-client
  icon: icon.png
  version: 1.0.0
---

# wave-insights

Análisis comparativos y detección de señales a partir del modelo semántico. Usa queries batch para minimizar latencia y costo, y el agente calcula variaciones, rankings y outliers del lado cliente.

## Reglas transversales (obligatorias en toda respuesta)

- **Modo por defecto: lenguaje de negocio.** Titular + tabla + bullets de hallazgos. Nada de JSON ni nombres técnicos.
- **Exploración técnica bajo pedido.** Si el usuario pregunta cómo se calcula un número del análisis, usa `wave_schema(detail="full", field=<measure>)` y, si quiere ver el SQL, repite la `wave_semantic_query` con `include_sql: "always"`.
- **No hay SQL arbitrario ni YAML crudo.** Si pide cross-check contra raw data, propón en su lugar desglosar por una dimensión relevante con `wave_semantic_query` adicional.
- **Anomalía detectada**: ofrece proactivamente "¿quieres que desglose este valor por <dimensión> para entender de dónde viene?" y ejecuta una query segmentada.
- **Español neutro obligatorio.** Prohibido voseo (vos / tenés / podés / querés / sos) y regionalismos (acá / allá / che). Tuteo neutro o impersonal: "tú puedes", "tú tienes", "aquí".

## Workflow

### 1. Bootstrap

```
wave_whoami
wave_list_semantic_layers
wave_schema({ semanticLayerId, detail: "names" })
```

### 2. Diseñar el batch

Construye 2–10 queries relacionadas. Patrones típicos:

**Comparar dos periodos**:
```
wave_semantic_query({
  dataSourceId,
  semanticLayerId,
  queries: [
    { cube, measures: ["monto_total"], filters: [{ dimension: "fecha", operator: "between", values: ["2026-01-01","2026-03-31"] }] },
    { cube, measures: ["monto_total"], filters: [{ dimension: "fecha", operator: "between", values: ["2025-01-01","2025-03-31"] }] }
  ]
})
```

**Top-N + tendencia**:
```
queries: [
  { cube, measures: ["monto_total"], dimensions: ["vendedor"], orderBy: "monto_total DESC", limit: 10,
    filters: [{ dimension: "fecha", operator: "between", values: [...] }] },
  { cube, measures: ["monto_total"], dimensions: ["fecha"], granularity: "month",
    filters: [{ dimension: "fecha", operator: "between", values: [...] }] }
]
```

Reglas:
- Ejecuta el batch en **una sola llamada** con `queries: [...]`. Nunca hagas N llamadas secuenciales.
- Límite por batch: 20 queries. Si necesitas más, divídelas en batches consecutivos.
- No uses `include_sql: "always"` en las queries del análisis; solo bajo pedido del usuario.

### 3. Calcular variaciones del lado cliente

Con los resultados en mano, calcula:
- % cambio vs baseline (periodo anterior, mismo periodo año pasado, promedio).
- Ranking y concentración (ej. "los 3 primeros explican el 60% del total").
- Outliers: valores a más de 2 desviaciones estándar del promedio.

No emitas más queries para esto; los datos ya están.

### 4. Presentar el análisis

Estructura:
1. **Titular** (una línea): "Ventas Q1 subieron 12% vs Q1 2025, impulsadas por la región Centro."
2. **Tabla compacta** con los números clave.
3. **3–5 hallazgos** en bullets, cada uno con el dato de respaldo.
4. **Cierre**: "¿Quieres que desglose por vendedor o que compare contra otro periodo?"

### 5. Profundización reactiva

Cuando el usuario acepta desglosar o detectas un outlier:
```
wave_semantic_query({ cube, measures, dimensions: [<dimensión_segmentación>], filters: [...] })
```
O si quiere entender cómo se calcula:
```
wave_schema({ semanticLayerId, detail: "full", field: "<cube>.<measure>" })
wave_semantic_query({ ...la_query, include_sql: "always" })
```

## Anti-patterns

- Hacer N queries secuenciales pudiendo ser un batch.
- Calcular variaciones emitiendo otra `wave_semantic_query` cuando los datos ya están.
- Dar conclusiones sin mostrar los números.
- Inferir causalidad ("las ventas bajaron **porque**..."); esa tarea es de `wave-recommendation`.
- Ofrecer "cuadrar con la tabla raw vía SQL libre"; propón un desglose dimensional en su lugar.

## Ejemplos de preguntas que resuelve

- "Compara las ventas de este trimestre vs el anterior."
- "Dame el top 10 de clientes del último mes."
- "Muéstrame la tendencia mensual de margen del último año."
- "¿Qué vendedor tuvo la mayor caída vs el mes pasado?"
- "¿Hay algo raro en las ventas de la semana pasada?"
