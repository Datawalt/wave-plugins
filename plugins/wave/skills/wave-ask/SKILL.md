---
name: wave-ask
description: Responde preguntas de negocio en lenguaje natural sobre los modelos de Wave. Cubre tres modos según la forma de la pregunta - cerrado (un número puntual), comparativo (tendencias, top-N, variaciones, outliers) y prescriptivo (recomendaciones accionables priorizadas). Úsalo cuando el usuario pregunte "cuánto vendí", "compará Q1 vs Q1", "dame el top 10", "qué pasó con X métrica", "dónde debería enfocar", "qué oportunidades hay", o cualquier consulta sobre sus datos.
metadata:
  mcp-server: wave-client
  icon: icon.png
  version: 2.0.0
---

# wave-ask

Punto de entrada único para conversación basada en datos sobre la capa semántica de Wave. La skill traduce una pregunta del usuario a una o más `wave_semantic_query` y devuelve la respuesta en lenguaje de negocio.

Post-ADR-0017 PR4: esta skill consolida tres skills previas (`wave-ask`, `wave-insights`, `wave-recommendation`) en un único entry point con tres modos enseñados por system prompt. El routing entre modos lo decide la skill en función del **shape de la pregunta** y el **shape del modelo**, no de una taxonomía cerrada de dominios (per ADR-0016 — el motor Wave es agnóstico al dominio).

## Reglas transversales (obligatorias en toda respuesta)

- **Modo por defecto: lenguaje de negocio.** Respuestas en prosa + tabla, con labels humanos. IDs internos (`dataSourceId`, `semanticLayerId`, nombres técnicos de cubes, claves como `cube.measure_key`) no aparecen salvo que el usuario los pida explícitamente.
- **Exploración técnica bajo pedido, siempre dentro del modelo semántico.** Si el usuario pide "cómo se calcula esto", "muéstrame el SQL que genera este número", "quiero entender la lógica", o investiga un valor anómalo, puedes: (a) mostrar el SQL generado usando `include_sql: "always"` en `wave_semantic_query`; (b) mostrar la definición técnica de la medida con `wave_schema(detail="full", field=<measure>)` (label, SQL base, filtros, tipo de agregación); (c) probar otra segmentación con `wave_semantic_query` para aislar la anomalía; (d) inspeccionar la distribución de una dimensión con `wave_profile(target="dimension")`.
- **No hay SQL arbitrario ni YAML crudo del modelo.** Las herramientas `wave_run_sql` y `wave_export` no están disponibles desde el plugin cliente. Si el usuario pide "dame acceso SQL libre" o "muéstrame el YAML", responde que eso queda fuera del alcance de la consola cliente y sugiere la alternativa (desglosar con `wave_semantic_query`, inspeccionar con `wave_schema(detail="full")`, o solicitar apoyo al equipo Datawalt).
- **Anomalía detectada**: ofrece proactivamente "¿quieres que desglose este número por <dimensión> para entender de dónde viene?" y ejecuta una query de segmentación. NO ofrezcas cross-check contra raw SQL.
- Traduce a labels de negocio usando `wave_schema(detail="names")` cuando presentas resultados, aunque internamente hayas visto los nombres técnicos.
- **Honestidad analítica.** Si los datos no respaldan una conclusión, dilo. No inventes causalidad ni extrapoles fuera de la evidencia. Especialmente importante en modo prescriptivo.
- **Español neutro obligatorio.** Prohibido voseo (vos / tenés / podés / querés / sos) y regionalismos (acá / allá / che). Usa tuteo neutro o formas impersonales: "puedes", "tienes", "quieres", "aquí". Esta regla aplica a toda respuesta al usuario y a los textos internos de la skill (ejemplos, anti-patterns).

## Routing por modo

Antes de cualquier query, clasifica la pregunta por su **shape**, no por el dominio del modelo. El mismo modelo (retail, educación, IoT, hospitalidad, etc.) puede recibir preguntas en cualquiera de los 3 modos.

| Shape de la pregunta | Modo | Ejemplo |
|---|---|---|
| "¿Cuánto X?" / "¿Cuántos Y?" / valor puntual sin baseline implícito | **Cerrado** | "¿Cuánto vendí en marzo?" — retail. "¿Cuántas inscripciones hubo este semestre?" — educación. |
| "Compará X vs Y" / "top N" / "qué cambió" / "tendencia" / "mejor / peor" / "outlier" | **Comparativo** | "Compará la ocupación de Q1 vs Q1 2025" — hospitalidad. "Top 10 sensores con más fallas el último mes" — IoT. |
| "¿Qué debería hacer?" / "¿Dónde enfocar?" / "recomiéndame" / "qué oportunidades" / "qué clientes en riesgo" | **Prescriptivo** | "¿Dónde debería enfocar al equipo comercial este mes?" — retail. "¿Qué cursos vale la pena expandir?" — educación. |

Si la pregunta es ambigua, pregunta antes de decidir el modo: "¿Quieres solo el número, una comparación contra otro periodo, o recomendaciones de qué hacer?"

## Workflow

### 0. Bootstrap (una vez por sesión)

```
wave_whoami
wave_list({ target: "semantic_layers" })
```

- Si hay un solo modelo disponible, úsalo.
- Si hay varios, pregunta al usuario en lenguaje de negocio: "Tienes estos modelos disponibles: A, B y C. ¿Sobre cuál quieres preguntar?"

Guarda `dataSourceId` y `semanticLayerId` para el resto de la conversación. Si el usuario cambia de modelo, vuelve a este paso.

### 1. Descubrir el modelo

```
wave_schema({ semanticLayerId, detail: "names" })
```

Te devuelve la lista de cubes con sus dimensiones y medidas. Úsala para mapear la pregunta del usuario a nombres técnicos. Si hay ambigüedad (p. ej. "ingresos" podría ser bruto o neto), pregunta al usuario antes de ejecutar.

**Lee el campo `description` del modelo** que devuelve este mismo `wave_schema` (en cualquier `detail`). Es el README del modelo: explica qué representa, cómo se relacionan los cubes y reglas de negocio transversales (ej. "las medidas con prefijo `gross_` incluyen devoluciones"). Es el contexto que distingue elegir bien una medida vs adivinar por el nombre.

Si hay 2+ medidas con nombres similares (`gross_revenue` vs `net_revenue`), llama además `wave_schema({ detail: "full", field: "<cube>.<measure>" })` antes de elegir y lee la `description` específica de cada una.

Para preguntas cross-cube (combinan info de dos áreas), `wave_schema({ detail: "join_path", fromCube, toCube })` explica cómo conectan.

### 2. Ejecutar según el modo

#### Modo cerrado (pregunta puntual)

Una sola query, una respuesta numérica.

```
wave_semantic_query({
  dataSourceId,
  semanticLayerId,
  cube: "<cube>",
  measures: ["<medida>"],
  dimensions: ["<dim_opcional>"],
  granularity: "month",            // si el corte es temporal
  filters: [
    { dimension: "<dim>", operator: "between", values: ["2026-01-01","2026-01-31"] }
  ],
  limit: 100
})
```

Reglas:
- **No** uses `include_sql: "always"` por defecto. Solo si el usuario lo pide.
- Si la pregunta no trae periodo, pregunta al usuario antes de asumir uno.
- Si el resultado sale vacío, explica al usuario por qué puede pasar (filtros demasiado estrictos, el periodo no tiene data) y propón una consulta menos restrictiva.
- **Refs cross-cube**: si una dim/measure pertenece a otro cube relacionado (p.ej. `sucursales.region` desde un fact de ventas), basta con escribir `targetCube.field` en `dimensions[]` o `measures[]`. El motor encuentra el join path automáticamente, incluso multi-hop. No enumeres hops manualmente — `cubeA.cubeB.field` no es sintaxis válida; usa `cubeB.field` directo y el motor hace el resto.

#### Modo comparativo (tendencias, top-N, variaciones, outliers)

Batch de 2-10 queries relacionadas en **una sola llamada** con `queries: [...]`. Nunca hagas N llamadas secuenciales — el motor optimiza la parse compartida cuando vienen juntas.

**Comparar dos periodos** (ejemplo retail):
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

**Top-N + tendencia** (ejemplo IoT — sensores con más eventos):
```
queries: [
  { cube, measures: ["eventos_count"], dimensions: ["sensor"], orderBy: "eventos_count DESC", limit: 10,
    filters: [{ dimension: "fecha", operator: "between", values: [...] }] },
  { cube, measures: ["eventos_count"], dimensions: ["fecha"], granularity: "month",
    filters: [{ dimension: "fecha", operator: "between", values: [...] }] }
]
```

**Calcular variaciones del lado cliente** con los resultados en mano:
- % cambio vs baseline (periodo anterior, mismo periodo año pasado, promedio).
- Ranking y concentración (ej. "los 3 primeros explican el 60% del total").
- Outliers: valores a más de 2 desviaciones estándar del promedio.

No emitas más queries para esto; los datos ya están. Límite por batch: 20 queries.

#### Modo prescriptivo (recomendaciones accionables)

Antes de ejecutar, **clarifica la pregunta de negocio** si es vaga ("dame recomendaciones"). Pregunta:
- ¿Cuál es la métrica de éxito (ingresos, ocupación, retención, cobertura)?
- ¿Qué ventana de tiempo te importa (mes, trimestre)?
- ¿Qué palanca tienes (equipo, pricing, mix, capacidad)?

No dispares queries hasta que quede claro. Después construye un batch de diagnóstico que responda 4 ejes:

- **Magnitud**: cuánto es el número hoy.
- **Baseline**: contra qué comparar (periodo anterior, mismo periodo año pasado, promedio).
- **Segmentación**: desglose por la dimensión candidata a accionar (equipo, canal, región, categoría, cliente, sede, curso, sensor).
- **Concentración**: distribución (¿Pareto, plano, larga cola?).

Una sola llamada batch con 3-8 queries relacionadas.

### 3. Presentar la respuesta

#### Modo cerrado

- Una frase de resumen con el número principal.
- Tabla Markdown solo si hay más de una fila, con headers humanos.
- Formato de cifras con separador de miles; moneda/unidad si el contexto lo indica.

#### Modo comparativo

Estructura:
1. **Titular** (una línea): "Las inscripciones de Q1 subieron 12% vs Q1 2025, impulsadas por el campus Centro." (educación)
2. **Tabla compacta** con los números clave.
3. **3-5 hallazgos** en bullets, cada uno con el dato de respaldo.
4. **Cierre**: "¿Quieres que desglose por <dim> o que compare contra otro periodo?"

#### Modo prescriptivo

Estructura fija de 4 secciones en este orden:

**Diagnóstico** (3-4 líneas con los hechos clave, con cifras).
> En Q1 2026 las reservas fueron 4.200, +12% vs Q1 2025. El segmento Boutique creció +28% y explicó 45% del total. Resort cayó -8%. (hospitalidad)

**Hipótesis** (1-3 interpretaciones de lo observado, **marcadas como hipótesis**, no como hechos).
> - El crecimiento de Boutique podría estar relacionado con la apertura del nuevo canal de partners en enero (hipótesis, no validable con estos datos).
> - La caída en Resort podría deberse a estacionalidad o pérdida de un cliente corporate grande.

**Recomendaciones** (2-5 bullets, priorizadas, cada una con el dato que la respalda y la acción concreta).
> 1. **Reforzar Boutique.** Creció +28% y representa 45% del total. Acción: reasignar X% del presupuesto comercial de Resort a Boutique por 60 días.
> 2. **Investigar Resort.** Caída -8% en Q1 con cliente Y pasando de 12% a 3% del mix. Acción: reunión con Y la próxima semana.

**Qué más analizar**.
> - Descomposición de Resort por cliente y producto.
> - Curva mensual de Boutique para confirmar si es tendencia o rebote puntual.

### 4. Modo audit (transversal, bajo pedido)

Cuando el usuario diga "cómo se calcula", "qué SQL genera esto", "quiero ver la definición":

```
wave_schema({ semanticLayerId, detail: "full", field: "<cube>.<measure>" })
wave_semantic_query({ ...misma_query, include_sql: "always" })
```

Muestra: el label oficial de la medida, su SQL base, los filtros que aplica la definición, y el SQL compilado de la query. Explica en una línea qué hace cada parte. No conviertas esto en el modo por defecto — es bajo pedido.

## Anti-patterns

- Ejecutar `wave_semantic_query` sin haber leído el schema antes (termina en "cube no existe").
- Devolver JSON crudo de la respuesta.
- Mostrar `medida_key` o `cube.medida_key` en la respuesta al usuario final.
- Asumir un `granularity` sin preguntar si el usuario no fue claro.
- Hacer N queries secuenciales en modo comparativo o prescriptivo pudiendo ser un batch.
- Calcular variaciones emitiendo otra `wave_semantic_query` cuando los datos ya están.
- Dar conclusiones sin mostrar los números.
- **Afirmar causalidad** en modo comparativo ("las ventas bajaron **porque**..."); eso es modo prescriptivo, y aún ahí se marca como hipótesis.
- Recomendar acciones que el rol del usuario no puede ejecutar.
- Más de 5 recomendaciones — diluye las decisiones.
- Pedir perdón o excusarte por no tener SQL libre; solo responde que la consola cliente no expone esa capacidad y ofrece la alternativa dentro del modelo.

## Ejemplos de preguntas que resuelve

**Modo cerrado:**
- "¿Cuánto vendí en marzo?" (retail)
- "¿Cuántos alumnos se inscribieron este semestre?" (educación)
- "¿Cuál fue el uptime promedio del último mes?" (IoT)
- "¿Cuál fue la ocupación promedio en abril?" (hospitalidad)
- "¿Cómo se calcula este número?" → modo audit.

**Modo comparativo:**
- "Compará las ventas de este trimestre vs el anterior." (retail)
- "Dame el top 10 de cursos con mayor matrícula." (educación)
- "Muéstrame la tendencia mensual de fallas del último año." (IoT)
- "¿Qué hotel tuvo la mayor caída de reservas vs el mes pasado?" (hospitalidad)
- "¿Hay algo raro en las ventas de la semana pasada?"

**Modo prescriptivo:**
- "Dame recomendaciones para mejorar las ventas del próximo trimestre."
- "¿Dónde debería enfocar al equipo comercial este mes?"
- "¿Qué cursos vale la pena expandir?" (educación)
- "¿Qué clientes están en riesgo?" (retail)
- "Dadas las tendencias, ¿qué debería hacer con el mix de habitaciones?" (hospitalidad)
