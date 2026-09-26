# Semana 7 · Consultas SQL y pandas — Línea de empaque L2 (Café)

**Caso:** planta agroindustrial del Huila que tuesta y empaca café. Se sigue trabajando sobre el modelo de la línea de empaque L2 (Semana 6): equipos, componentes y paradas no programadas, con el fin de identificar qué componente genera más tiempo de parada y apoyar la estimación de fallas en las próximas 72 horas.

Este ejercicio usa el mismo dataset relacional del ERD anterior (`EQUIPO`, `COMPONENTE`, `EQUIPO_COMPONENTE`, `PARADA`), con datos de ejemplo para poder ejecutar y explicar las consultas.

---

## 1. Dataset de trabajo

### Tabla `EQUIPO`

| equipo_id | nombre | tipo | linea |
|---|---|---|---|
| 1 | Tostadora T1 | Tueste | L2 |
| 2 | Dosificadora D1 | Dosificado | L2 |
| 3 | Selladora S1 | Sellado | L2 |
| 4 | Empacadora E1 | Empacado | L2 |

### Tabla `COMPONENTE`

| componente_id | nombre | tipo | vida_util_horas |
|---|---|---|---|
| 1 | Motor principal | Motor | 8000 |
| 2 | Banda transportadora | Banda | 5000 |
| 3 | Válvula dosificadora | Válvula | 3000 |
| 4 | Sensor de peso | Sensor | 10000 |
| 5 | Resistencia de sellado | Resistencia | 4000 |

### Tabla `PARADA`

| parada_id | equipo_id | componente_id | fecha_inicio | fecha_fin | tipo | causa_raiz |
|---|---|---|---|---|---|---|
| 1 | 1 | 1 | 2026-09-01 06:10 | 2026-09-01 07:40 | No programada | Sobrecalentamiento motor |
| 2 | 2 | 3 | 2026-09-03 14:20 | 2026-09-03 14:50 | No programada | Válvula atascada |
| 3 | 3 | 5 | 2026-09-05 09:00 | 2026-09-05 09:15 | Programada | Mantenimiento preventivo |
| 4 | 1 | 2 | 2026-09-10 22:05 | 2026-09-11 00:30 | No programada | Rotura de banda |
| 5 | 4 | 2 | 2026-09-12 11:00 | 2026-09-12 11:45 | No programada | Banda desalineada |
| 6 | 2 | 4 | 2026-09-15 08:00 | 2026-09-15 08:20 | Programada | Calibración de sensor |

`componente_id = 2` (banda transportadora) aparece en dos equipos distintos (1 y 4), lo que ilustra la relación N:M `EQUIPO_COMPONENTE` definida en la Semana 6.

---

## 2. Consulta SQL con `WHERE`

**Objetivo:** aislar únicamente las paradas *no programadas* que superaron 30 minutos de duración, porque son las que realmente impactan el objetivo del proyecto (predecir fallas), a diferencia de una parada programada de mantenimiento.

```sql
SELECT
    parada_id,
    equipo_id,
    componente_id,
    fecha_inicio,
    fecha_fin,
    causa_raiz
FROM PARADA
WHERE tipo = 'No programada'
  AND TIMESTAMPDIFF(MINUTE, fecha_inicio, fecha_fin) > 30;
```

**Qué responde:** "¿Cuáles fueron las paradas no programadas realmente significativas (más de 30 minutos) registradas en la línea L2?". Filtra el ruido de micro-detenciones y deja solo los eventos que vale la pena analizar como candidatos a falla crítica.

---

## 3. Consulta SQL con `JOIN`

**Objetivo:** las tablas `PARADA`, `EQUIPO` y `COMPONENTE` solo guardan IDs (por normalización, ver Semana 6), así que se necesita un `JOIN` para poder leer la información en términos de negocio: nombre del equipo y nombre del componente causante, no simples números.

```sql
SELECT
    p.parada_id,
    e.nombre        AS equipo,
    c.nombre        AS componente_causante,
    p.fecha_inicio,
    p.fecha_fin,
    p.causa_raiz
FROM PARADA p
JOIN EQUIPO e
    ON p.equipo_id = e.equipo_id
LEFT JOIN COMPONENTE c
    ON p.componente_id = c.componente_id
WHERE p.tipo = 'No programada';
```

> Se usa `LEFT JOIN` con `COMPONENTE` porque a veces una parada no programada se registra sin que aún se haya identificado el componente causante (`componente_id` nulo); con `LEFT JOIN` esa parada no se pierde, solo aparece con el componente en blanco.

**Qué responde:** "¿Qué equipo específico se detuvo y qué componente fue el causante de cada parada no programada?". Convierte el modelo normalizado (con IDs) en un reporte legible que un supervisor de planta puede interpretar directamente.

---

## 4. Consulta SQL con `GROUP BY`

**Objetivo:** este es el punto clave del proyecto — identificar **qué componente concentra más tiempo perdido**, es decir, el candidato más probable a causar la próxima falla no programada en las siguientes 72 horas.

```sql
SELECT
    c.nombre AS componente,
    COUNT(*) AS total_paradas,
    SUM(TIMESTAMPDIFF(MINUTE, p.fecha_inicio, p.fecha_fin)) AS minutos_totales_perdidos
FROM PARADA p
JOIN COMPONENTE c
    ON p.componente_id = c.componente_id
WHERE p.tipo = 'No programada'
GROUP BY c.nombre
ORDER BY minutos_totales_perdidos DESC;
```

**Qué responde:** "¿Cuál componente ha generado más paradas y más minutos de línea detenida hasta ahora?". Con los datos de ejemplo, la **banda transportadora** acumula el mayor tiempo perdido (dos eventos, en dos equipos distintos), por lo que sería el primer candidato a vigilar de cerca en el modelo predictivo de fallas.

---

## 5. Equivalente en pandas (`groupby`)

Se reproduce exactamente la consulta del punto 4 (GROUP BY) usando pandas, partiendo de los mismos datos:

```python
import pandas as pd

# Datos de ejemplo equivalentes a las tablas SQL
paradas = pd.DataFrame({
    "parada_id":     [1, 2, 3, 4, 5, 6],
    "equipo_id":     [1, 2, 3, 1, 4, 2],
    "componente_id": [1, 3, 5, 2, 2, 4],
    "fecha_inicio":  pd.to_datetime([
        "2026-09-01 06:10", "2026-09-03 14:20", "2026-09-05 09:00",
        "2026-09-10 22:05", "2026-09-12 11:00", "2026-09-15 08:00"
    ]),
    "fecha_fin":     pd.to_datetime([
        "2026-09-01 07:40", "2026-09-03 14:50", "2026-09-05 09:15",
        "2026-09-11 00:30", "2026-09-12 11:45", "2026-09-15 08:20"
    ]),
    "tipo": [
        "No programada", "No programada", "Programada",
        "No programada", "No programada", "Programada"
    ],
})

componentes = pd.DataFrame({
    "componente_id": [1, 2, 3, 4, 5],
    "nombre": ["Motor principal", "Banda transportadora",
               "Válvula dosificadora", "Sensor de peso", "Resistencia de sellado"],
})

# 1. Filtrar solo paradas no programadas (equivalente al WHERE)
no_programadas = paradas[paradas["tipo"] == "No programada"].copy()

# 2. Calcular duración en minutos
no_programadas["minutos"] = (
    (no_programadas["fecha_fin"] - no_programadas["fecha_inicio"])
    .dt.total_seconds() / 60
)

# 3. Unir con componentes (equivalente al JOIN)
detalle = no_programadas.merge(componentes, on="componente_id")

# 4. Agrupar por componente (equivalente al GROUP BY)
resumen = (
    detalle.groupby("nombre")
    .agg(total_paradas=("parada_id", "count"),
         minutos_totales_perdidos=("minutos", "sum"))
    .sort_values("minutos_totales_perdidos", ascending=False)
    .reset_index()
)

print(resumen)
```

**Salida esperada (equivalente a la consulta SQL del punto 4):**

| nombre | total_paradas | minutos_totales_perdidos |
|---|---|---|
| Banda transportadora | 2 | 235.0 |
| Motor principal | 1 | 90.0 |
| Válvula dosificadora | 1 | 30.0 |

El resultado es el mismo que el de la consulta SQL con `GROUP BY`: pandas reemplaza el `JOIN` con `.merge()`, el `WHERE` con un filtrado booleano sobre el DataFrame, y el `GROUP BY ... SUM/COUNT` con `.groupby().agg()`.

---

## 6. Resumen de qué responde cada consulta

| Consulta | Cláusula clave | Pregunta de negocio que responde |
|---|---|---|
| 1 | `WHERE` | ¿Qué paradas no programadas fueron realmente significativas (>30 min)? |
| 2 | `JOIN` | ¿Qué equipo y qué componente causaron cada parada no programada, en términos legibles? |
| 3 | `GROUP BY` | ¿Qué componente acumula más tiempo de parada y es el principal candidato a fallar de nuevo? |
| pandas | `groupby` | Reproduce la pregunta de la consulta 3 fuera de la base de datos, útil para integrarla directo en el pipeline de análisis/modelo predictivo en Python. |
