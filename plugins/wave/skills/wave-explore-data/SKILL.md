---
name: wave-explore-data
description: Ayuda al usuario a descubrir qué datos, métricas y dashboards están disponibles en su empresa. Úsalo cuando el usuario pregunte "qué puedo consultar", "qué datos tengo", "qué métricas hay", "qué dashboards existen" o quiera explorar antes de preguntar.
metadata:
  mcp-server: wave-client
  icon: icon.png
  version: 2.0.0
---

# wave-explore-data

Descubrimiento del catálogo de datos disponible para el usuario: modelos semánticos, dashboards, cubes, métricas y dimensiones, traducidos a lenguaje de negocio. Skill agnóstica al dominio del modelo (per ADR-0016) — sirve igual para retail, educación, IoT, hospitalidad u otras verticales.

## Reglas transversales (obligatorias en toda respuesta)

- **Modo por defecto: lenguaje de negocio.** Respuestas en prosa + tabla, con labels humanos. IDs internos no aparecen salvo que el usuario los pida.
- **Exploración técnica bajo pedido, siempre dentro del modelo semántico.** Si el usuario pide la definición técnica de una medida, usa `wave_schema(detail="full", field=...)`; si quiere ver la distribución de una dimensión, usa `wave_profile(target="dimension")`; si quiere inspeccionar data cruda, usa `wave_preview`.
- **No hay SQL arbitrario ni YAML crudo.** Si piden "exportar el modelo" o "acceso SQL libre", responde que esas capacidades no están disponibles aquí.
- **Español neutro obligatorio.** Prohibido voseo (vos / tenés / podés / querés / sos) y regionalismos (acá / allá / che). Usa tuteo neutro o formas impersonales: "tú puedes", "tú tienes", "aquí".

## Workflow

### 1. Bootstrap

```
wave_whoami
wave_list({ target: "dashboards" })
wave_list({ target: "semantic_layers" })
```

Presenta al usuario un resumen: "Tienes X modelos de datos y Y dashboards. Estos son los principales: ...". Nombres humanos, sin IDs.

### 2. Catálogo del modelo elegido

Cuando el usuario se interese por un modelo específico:

```
wave_schema({ semanticLayerId, detail: "names" })
```

Responde con una lista por área. Ejemplos:
- **Retail**: "El modelo tiene los cubes Pedidos, Clientes y Productos. En Pedidos puedes medir: monto total, cantidad de pedidos, ticket promedio. Lo puedes cortar por: mes, canal, región."
- **Educación**: "El modelo tiene los cubes Matrículas, Alumnos y Cursos. En Matrículas puedes medir: cantidad de inscripciones, tasa de aprobación. Lo puedes cortar por: semestre, campus, programa."
- **IoT**: "El modelo tiene los cubes Eventos, Dispositivos y Sensores. En Eventos puedes medir: count, uptime promedio, MTBF. Lo puedes cortar por: día, dispositivo, tipo de evento."
- **Hospitalidad**: "El modelo tiene los cubes Reservas, Huéspedes y Habitaciones. En Reservas puedes medir: ocupación, ADR, RevPAR. Lo puedes cortar por: noche, segmento, categoría de habitación."

### 3. Detalle de un cube

```
wave_schema({ semanticLayerId, detail: "full", cube: "<cube>" })
```

Muestra medidas y dimensiones con sus labels y, cuando estén, sus descripciones. No muestres el SQL base salvo que el usuario pregunte "cómo se calcula esto".

### 4. Ver cómo se ve la data (opcional)

Para inspeccionar filas reales del cube modelado:
```
wave_preview({ target: "cube", dataSourceId, semanticLayerId, cube: "<cube>", limit: 10 })
```

Para inspeccionar una tabla cruda del data source (power user):
```
wave_list({ target: "tables", dataSourceId })                        // descubrir
wave_preview({ target: "table", dataSourceId, tableName: "<schema.tabla>", limit: 10 })
```

Presenta como tabla Markdown con headers humanos.

### 5. Pedir un campo de un cube relacionado

Cuando el usuario pida un campo de otro cube (ej. "inscripciones por campus" donde `inscripciones` y `campus` son cubes distintos en un modelo de educación; o "eventos por dispositivo" en IoT), basta con escribir `targetCube.field` en `dimensions[]` o `measures[]`. El motor encuentra el JOIN automáticamente:

```
wave_semantic_query({
  cube: "eventos",
  measures: ["uptime_avg"],
  dimensions: ["dispositivos.tipo"]   // cross-cube — JOIN automático (ejemplo IoT)
})
```

El motor incluso resuelve multi-hop: `dimensions: ["region.nombre"]` desde un fact con cadena `fact → entidad → region` declarada en YAML traversa los 2 hops sin que tengas que enumerarlos. Si el motor responde "no join path", indica que la relación no está declarada en el modelo — pide ayuda al equipo Datawalt.

Si la pregunta cruza áreas y querés entender la conexión antes de consultar, `wave_schema({ detail: "join_path", fromCube, toCube })` explica los hops.

### 6. Distribución de una dimensión

```
wave_profile({ target: "dimension", dataSourceId, semanticLayerId, cube: "<cube>", dimension: "<dim>" })
```

Úsalo cuando el usuario pregunte "qué valores toma X" o "cuáles son los principales <categoría>". Ejemplos: "¿qué categorías de producto hay?" (retail), "¿qué tipos de habitación existen?" (hospitalidad), "¿qué programas se ofrecen?" (educación). Presenta el top-N en prosa, no como JSON.

### 7. Dashboards existentes

```
wave_list({ target: "dashboards" })
```

Lista por nombre. Si el usuario pregunta qué contiene un dashboard, nómbraselo y sugiere: "Puedo ayudarte a reproducir las mismas preguntas con `wave-ask` si quieres profundizar".

## Anti-patterns

- Listar IDs numéricos como si fueran datos útiles para el usuario.
- Ofrecer "exportar el modelo" o "dame el YAML": esas capacidades no existen en esta consola.
- Volcar JSON crudo del schema o del preview.
- Llamar a `wave_list({ target: "tables" })` sin propósito claro — es más ruido que señal para un usuario de negocio.

## Ejemplos de preguntas que resuelve

- "¿Qué puedo preguntar sobre este modelo?"
- "¿Qué dashboards tengo disponibles?"
- "¿Qué dimensiones tiene el modelo?"
- "¿Qué categorías de producto hay?" (retail)
- "¿Qué tipos de sensor están reportando?" (IoT)
- "¿Cuántos programas distintos están activos?" (educación)
- "Muéstrame cómo se ven los datos de reservas." (hospitalidad)
