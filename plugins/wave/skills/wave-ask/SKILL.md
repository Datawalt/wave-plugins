---
name: wave-ask
description: Responde preguntas de negocio en lenguaje natural consultando los modelos de datos de Wave. Úsalo cuando el usuario pregunte "cuánto vendí", "cuántos clientes", "qué pasó con X métrica", o cualquier consulta puntual sobre sus datos.
metadata:
  mcp-server: wave-client
  icon: icon.png
  version: 1.0.0
---

# wave-ask

Pregunta y respuesta sobre los datos de la empresa usando la capa semántica de Wave. La skill traduce una pregunta del usuario a una `wave_semantic_query` y devuelve la respuesta en lenguaje de negocio.

## Reglas transversales (obligatorias en toda respuesta)

- **Modo por defecto: lenguaje de negocio.** Respuestas en prosa + tabla, con labels humanos. IDs internos (`dataSourceId`, `semanticLayerId`, nombres técnicos de cubes, claves como `cube.measure_key`) no aparecen salvo que el usuario los pida explícitamente.
- **Exploración técnica bajo pedido, siempre dentro del modelo semántico.** Si el usuario pide "cómo se calcula esto", "muéstrame el SQL que genera este número", "quiero entender la lógica", o investiga un valor anómalo, puedes: (a) mostrar el SQL generado usando `include_sql: "always"` en `wave_semantic_query`; (b) mostrar la definición técnica de la medida con `wave_schema(detail="full", field=<measure>)` (label, SQL base, filtros, tipo de agregación); (c) probar otra segmentación con `wave_semantic_query` para aislar la anomalía; (d) inspeccionar la distribución de una dimensión con `wave_profile(target="dimension")`.
- **No hay SQL arbitrario ni YAML crudo del modelo.** Las herramientas `wave_run_sql` y `wave_export_semantic_layer` no están disponibles. Si el usuario pide "dame acceso SQL libre" o "muéstrame el YAML", responde que eso queda fuera del alcance de la consola cliente y sugiere la alternativa (desglosar con `wave_semantic_query`, inspeccionar con `wave_schema(detail="full")`, o solicitar apoyo al equipo Datawalt).
- **Anomalía detectada**: ofrece proactivamente "¿quieres que desglose este número por <dimensión> para entender de dónde viene?" y ejecuta una query de segmentación. NO ofrezcas cross-check contra raw SQL.
- Traduce a labels de negocio usando `wave_schema(detail="names")` cuando presentas resultados, aunque internamente hayas visto los nombres técnicos.
- **Español neutro obligatorio.** Prohibido voseo (vos / tenés / podés / querés / sos) y regionalismos (acá / allá / che). Usa tuteo neutro o formas impersonales: "puedes", "tienes", "quieres", "aquí". Esta regla aplica a toda respuesta al usuario y a los textos internos de la skill (ejemplos, anti-patterns).

## Workflow

### 1. Bootstrap (una vez por sesión)

```
wave_whoami
```
Confirma la empresa y el email del token. Si el usuario abre una sesión nueva, ejecútalo de nuevo.

### 2. Elegir modelo

```
wave_list_semantic_layers
```
- Si hay un solo modelo disponible, úsalo.
- Si hay varios, pregunta al usuario en lenguaje de negocio: "Tienes estos modelos disponibles: Ventas, Inventario y Comercial. ¿Sobre cuál quieres preguntar?"

Guarda `dataSourceId` y `semanticLayerId` para el resto de la conversación.

### 3. Descubrir métricas y dimensiones disponibles

```
wave_schema({ semanticLayerId, detail: "names" })
```
Te devuelve la lista de cubes con sus dimensiones y medidas. Úsala para mapear la pregunta del usuario a nombres técnicos. Si hay ambigüedad (p. ej. "ventas" podría ser monto total o unidades), pregunta al usuario antes de ejecutar.

**Lee el campo `description` del modelo** que devuelve este mismo `wave_schema` (en cualquier `detail`). Es el README del modelo: explica qué representa, cómo se relacionan los cubes y reglas de negocio transversales (ej. "las medidas con prefijo `gross_` incluyen devoluciones"). Es el contexto que distingue elegir bien una medida vs adivinar por el nombre.

Si hay 2+ medidas con nombres similares (`gross_revenue` vs `net_revenue`), llama además `wave_schema({ detail: "full", field: "<cube>.<measure>" })` antes de elegir y lee la `description` específica de cada una.

### 4. Ejecutar la pregunta

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
- **Refs cross-cube**: si una dim/measure pertenece a otro cube relacionado (p.ej. `sucursales.nombre` desde un fact de ventas), basta con escribir `targetCube.field` en `dimensions[]` o `measures[]`. El motor encuentra el join path automáticamente, incluso multi-hop. No enumeres hops manualmente — `cubeA.cubeB.field` no es sintaxis válida; usa `cubeB.field` directo y el motor hace el resto.

### 5. Presentar la respuesta

- Una frase de resumen con el número principal.
- Tabla Markdown solo si hay más de una fila, con headers humanos.
- Formato de cifras con separador de miles; moneda si el contexto lo indica.

### 6. Modo audit (solo bajo pedido)

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
- Pedir perdón o excusarte por no tener SQL libre; solo responde que la consola cliente no expone esa capacidad y ofrece la alternativa dentro del modelo.

## Ejemplos de preguntas que resuelve

- "¿Cuánto vendí en marzo?"
- "¿Cuántos clientes nuevos tuvimos el último trimestre?"
- "¿Cuál fue el margen del mes pasado?"
- "¿Cómo se calcula este número?" → modo audit.
- "¿Este total incluye devoluciones?" → `wave_schema(detail="full", field=...)` para mostrar la definición de la medida.
