# Taller de Análisis de Datos

**Autora:** Paula Ximena Romero

**Temas:** series de tiempo (Python/pandas), visualización de datos, optimización de consultas SQL, medidas DAX (Power BI) y requisito en inglés.

---

## Contenido

1. [Introducción](#introducción)
2. [Problema 1 — Serie de tiempo](#problema-1--serie-de-tiempo)
3. [Problema 2 — Visualización](#problema-2--visualización)
4. [Problema 3 — Optimización y métricas](#problema-3--optimización-y-métricas)
5. [English requirement](#english-requirement)
6. [Conclusiones](#conclusiones)

---

## Introducción

Este documento reúne la solución de los tres problemas del taller. En cada uno se indica el **objetivo**, el **procedimiento**, el **código** (comentado línea por línea cuando es necesario), el **resultado** y la **justificación** de las decisiones tomadas.

Como el enunciado no incluye una base de datos, el Problema 1 trabaja con una **serie de ejemplo generada con código** (ventas diarias simuladas durante dos años). El procedimiento es idéntico al que se aplicaría a una serie real cargada desde un archivo CSV; en el código se indica la línea que se cambiaría.

Librerías necesarias para ejecutar el código: `pandas`, `numpy` y `matplotlib` (se instalan con `pip install pandas numpy matplotlib`).

---

## Problema 1 — Serie de tiempo

### 1.1 Objetivo

Cargar una serie con la **fecha como índice**, **agregarla por periodo** con `resample`, calcular una **media móvil** e **identificar la tendencia y la estacionalidad**.

### 1.2 Conceptos previos

- **Serie de tiempo:** conjunto de datos medidos en momentos sucesivos y ordenados cronológicamente (por ejemplo, ventas por día).
- **Tendencia:** dirección de largo plazo de la serie (creciente, decreciente o estable).
- **Estacionalidad:** patrón que se repite cada cierto periodo fijo (por ejemplo, cada año).
- **Ruido (componente irregular):** variación aleatoria que no se explica por la tendencia ni por la estacionalidad.
- **`resample`:** método de pandas que cambia la frecuencia de una serie con índice de fechas (de diario a mensual, por ejemplo) aplicando una función de agregación (`sum`, `mean`, etc.).
- **Media móvil:** promedio de los últimos *n* periodos que se va desplazando en el tiempo. Suaviza las oscilaciones de corto plazo y deja ver el comportamiento general.

### 1.3 Procedimiento

1. **Cargar los datos con la fecha como índice.** Con un archivo real se usaría `pd.read_csv("ventas.csv", parse_dates=["fecha"], index_col="fecha")`. Aquí se genera una serie diaria de 2 años con tres componentes: una tendencia creciente (`0.08 * t`), una estacionalidad anual (una onda seno de periodo 365 días) y ruido aleatorio.
2. **Agregar por periodo** con `resample("MS").sum()`, que suma las ventas diarias de cada mes (`MS` = inicio de mes).
3. **Calcular medias móviles** de 3 y de 12 meses con `rolling(window=n).mean()`.
4. **Graficar** la serie mensual y las dos medias móviles.
5. **Separar la estacionalidad**: se ajusta una recta a la serie mensual (la tendencia), se resta de los datos y se promedia lo que queda por mes del año.

### 1.4 Código

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

# 1. Serie de ejemplo con la fecha como índice (2 años, datos diarios)
np.random.seed(42)                                   # para que el resultado sea reproducible
fechas = pd.date_range("2024-01-01", "2025-12-31", freq="D")
t = np.arange(len(fechas))                           # 0, 1, 2, ... (número de día)
valores = (
    100                                              # nivel base
    + 0.08 * t                                       # tendencia creciente
    + 15 * np.sin(2 * np.pi * t / 365)               # estacionalidad anual
    + np.random.normal(0, 5, len(t))                 # ruido aleatorio
)
serie = pd.Series(valores, index=fechas, name="ventas")

# 2. Agregar por periodo: suma mensual
mensual = serie.resample("MS").sum()

# 3. Medias móviles
mm3 = mensual.rolling(window=3).mean()               # ventana de 3 meses
mm12 = mensual.rolling(window=12).mean()             # ventana de 12 meses (un año completo)

# 4. Gráfico
plt.figure(figsize=(10, 5))
plt.plot(mensual, label="Ventas mensuales", marker="o")
plt.plot(mm3, label="Media móvil 3 meses")
plt.plot(mm12, label="Media móvil 12 meses", linewidth=2)
plt.title("Ventas mensuales y medias móviles")
plt.xlabel("Mes")
plt.ylabel("Ventas")
plt.legend()
plt.tight_layout()
plt.show()

# 5. Separar la estacionalidad
x = np.arange(len(mensual))
pendiente, intercepto = np.polyfit(x, mensual.values, 1)      # recta de tendencia
sin_tendencia = mensual - (pendiente * x + intercepto)        # datos sin tendencia
estacional = sin_tendencia.groupby(sin_tendencia.index.month).mean()
print("Aumento promedio por mes:", round(pendiente, 1))
print(estacional.round(0))
```

### 1.5 Resultado

Al ejecutar el código se obtiene un gráfico con tres líneas (ventas mensuales, media móvil de 3 meses y media móvil de 12 meses) y, en la consola, estos valores:

```
Media móvil de 12 meses:  dic-2024 ≈ 3.497   ->   dic-2025 ≈ 4.370
Aumento promedio por mes (pendiente): 56.8

Efecto estacional promedio por mes del año (ventas sin tendencia):
 1: +84    2: -8     3: +428   4: +326   5: +360   6: +46
 7: -22    8: -222   9: -442  10: -309  11: -280  12: +39
```

### 1.6 Interpretación

**Tendencia.** La media móvil de 12 meses pasa de aproximadamente **3.497** (diciembre de 2024) a **4.370** (diciembre de 2025), un aumento cercano al **25 %**. La recta de tendencia indica un crecimiento promedio de unas **57 unidades por mes**. Por tanto, la serie tiene una **tendencia creciente**.

**¿Por qué se usa una media móvil de 12 meses para ver la tendencia?** Porque una ventana de 12 meses cubre un ciclo estacional completo: los picos y valles del año se compensan entre sí y solo queda el comportamiento de largo plazo. La media móvil de 3 meses, en cambio, sigue más de cerca la serie y suaviza únicamente el ruido.

**Estacionalidad.** Después de quitar la tendencia, el promedio por mes muestra un patrón claro:

| Periodo | Efecto estacional (promedio) |
|---|---|
| Marzo a mayo | Por encima de lo normal (≈ +330 a +430) |
| Agosto a noviembre | Por debajo de lo normal (≈ −220 a −440) |
| Junio, julio, diciembre, enero, febrero | Cercano a cero |

Este patrón se repite cada año, y eso es lo que define la **estacionalidad**. Se ve en el gráfico como ondas que suben y bajan alrededor de la media móvil de 12 meses.

**Ruido.** Es la dispersión que queda alrededor del patrón; en el gráfico son las pequeñas diferencias de un mes a otro que no siguen ni la tendencia ni el ciclo anual.

> Nota: los valores numéricos corresponden a la serie simulada del ejemplo. Con una serie real el procedimiento es el mismo, pero las cifras cambian.

---

## Problema 2 — Visualización

### 2.1 Objetivo

Elegir el **gráfico adecuado** para tres preguntas distintas (evolución, comparación y proporción), justificar cada elección, **detectar un error de diseño** y proponer su corrección.

### 2.2 Elección del gráfico correcto

La regla general es que **el tipo de gráfico depende de la pregunta que se quiere responder**.

| Pregunta | Gráfico elegido | Justificación |
|---|---|---|
| **Evolución:** ¿cómo cambiaron las ventas a lo largo de los meses? | **Gráfico de líneas** (serie de tiempo) | La línea une los puntos en orden cronológico, por lo que se ve la dirección, los picos y los cambios en el tiempo. Barras o torta no transmiten bien la continuidad temporal. |
| **Comparación:** ¿qué producto vendió más? | **Gráfico de barras** | Cada categoría tiene su barra, y la longitud permite comparar magnitudes de un vistazo. Es más preciso que la torta para comparar. |
| **Proporción:** ¿qué porcentaje del total aporta cada línea de producto? | **Gráfico de torta** (solo con pocas categorías) | Muestra cómo se reparte un total entre sus partes. Funciona con pocas porciones (idealmente 5 o menos); con muchas se vuelve ilegible. |

### 2.3 Error de diseño detectado: eje Y truncado

**Descripción del error.** En un gráfico de barras, el eje vertical **no empieza en cero** (por ejemplo, empieza en 90). Con ventas de 95, 100 y 98, las diferencias reales son pequeñas (del orden de 5 %), pero las barras parecen muy distintas entre sí.

**Por qué es un error.** En un gráfico de barras, la **longitud de la barra representa el valor**. Si se corta la base, la proporción visual deja de corresponder a la proporción real y el lector saca una conclusión exagerada o falsa.

**Corrección.** Hacer que el eje Y empiece en **0**. Las barras vuelven a ser proporcionales y se ve que los tres productos tienen ventas muy parecidas.

```python
import matplotlib.pyplot as plt

productos = ["A", "B", "C"]
ventas = [95, 100, 98]

fig, ejes = plt.subplots(1, 2, figsize=(10, 4))

# Incorrecto: eje Y truncado
ejes[0].bar(productos, ventas, color="tab:red")
ejes[0].set_ylim(90, 102)
ejes[0].set_title("Incorrecto: eje Y truncado (empieza en 90)")
ejes[0].set_ylabel("Ventas (millones)")

# Correcto: eje Y desde 0
ejes[1].bar(productos, ventas, color="tab:blue")
ejes[1].set_ylim(0, 110)
ejes[1].set_title("Correcto: eje Y desde 0")
ejes[1].set_ylabel("Ventas (millones)")

plt.tight_layout()
plt.show()
```

### 2.4 Otro error frecuente: torta saturada

**Descripción.** Una torta con muchas categorías (por ejemplo 10 o más), muchos colores y porciones muy pequeñas.

**Por qué es un error.** El ojo humano compara mal los ángulos y las áreas; con muchas porciones no se distinguen ni se pueden leer las etiquetas.

**Corrección.** Usar un **gráfico de barras ordenado de mayor a menor**, o agrupar las categorías pequeñas en una sola llamada "Otros" y dejar la torta con pocas porciones.

### 2.5 Principio de diseño aplicado: *data-ink*

Según el principio *data-ink* (Edward Tufte), la mayor parte de la tinta de un gráfico debe representar datos. Por eso, en todos los gráficos conviene eliminar efectos 3D, fondos recargados y cuadrículas excesivas, y usar títulos, ejes y etiquetas claros.

---

## Problema 3 — Optimización y métricas

### 3.1 Objetivo

Reescribir una **consulta SQL lenta** aplicando buenas prácticas, explicar por qué mejora, crear **dos medidas DAX** y explicar la diferencia entre **medida** y **columna calculada**.

### 3.2 Consulta lenta (original)

```sql
SELECT *
FROM ventas
WHERE YEAR(fecha) = 2026;
```

**Problemas que tiene:**

1. **`SELECT *`** trae todas las columnas de la tabla aunque solo se necesiten algunas. Eso aumenta la lectura de disco, el uso de memoria y los datos enviados por la red.
2. **`YEAR(fecha)` en el `WHERE`** aplica una función a la columna. El motor debe calcular el año de **cada fila** antes de comparar, y como el índice guarda los valores originales de `fecha` (no el resultado de `YEAR(fecha)`), **no puede usarlo**. El resultado es un recorrido completo de la tabla (*full table scan*). Se dice que la condición no es *sargable*.

### 3.3 Consulta optimizada

```sql
-- 1) Índice sobre la columna usada en el filtro
CREATE INDEX idx_ventas_fecha ON ventas (fecha);

-- 2) Solo las columnas necesarias + filtro por rango sobre la columna sin modificar
SELECT id_venta, fecha, producto, monto
FROM ventas
WHERE fecha >= '2026-01-01'
  AND fecha <  '2027-01-01';
```

### 3.4 Por qué mejora

| Mejora aplicada | Explicación |
|---|---|
| **Columnas necesarias** en lugar de `*` | Se leen y transfieren menos datos. |
| **Filtro por rango** (`>=` y `<`) | La columna `fecha` queda sin funciones, así que el motor puede usar el índice. Usar `<` con el 1 de enero del año siguiente cubre todo el 2026 sin errores con horas o minutos. |
| **Índice en `fecha`** | Es una estructura ordenada (por ejemplo un árbol B) con punteros a las filas. El motor va directo al rango buscado en lugar de leer toda la tabla. |

El **resultado es el mismo** (las ventas de 2026), pero se usan menos tiempo, memoria y lecturas de disco. Eso es optimizar una consulta. La contrapartida del índice es que ocupa espacio extra y hace un poco más lentas las inserciones y actualizaciones.

### 3.5 Medidas DAX

Se asume un modelo con una tabla `Ventas` que tiene las columnas `Monto` y `Producto`.

```dax
Total Ventas = SUM ( Ventas[Monto] )
```

**Qué hace:** suma el monto de las ventas que estén visibles según los filtros activos (por ejemplo, un año o un producto elegido en un segmentador).

```dax
% del Total =
DIVIDE (
    [Total Ventas],
    CALCULATE ( [Total Ventas], ALL ( Ventas[Producto] ) )
)
```

**Qué hace:**

- `[Total Ventas]` (numerador): ventas del producto o contexto actual.
- `CALCULATE ( [Total Ventas], ALL ( Ventas[Producto] ) )` (denominador): ventas de **todos** los productos, porque `ALL` quita el filtro de producto.
- `DIVIDE` realiza la división y evita el error por división entre cero.

El resultado es el porcentaje que representa cada producto sobre el total.

### 3.6 Diferencia entre medida y columna calculada

| Característica | **Medida** | **Columna calculada** |
|---|---|---|
| **Cuándo se calcula** | Al momento de consultar, según el contexto de filtros actual (se calcula "al vuelo") | Al cargar o actualizar los datos |
| **Almacenamiento** | No se guarda en el modelo; solo se guarda la fórmula | Se guarda el valor de cada fila en la tabla (ocupa memoria) |
| **Resultado** | Cambia según los segmentadores, filtros y la visual | Es fijo para cada fila |
| **Uso típico** | Agregaciones: totales, promedios, porcentajes, comparaciones entre periodos | Valores por fila: categorías, banderas, textos, rangos |
| **Ejemplo** | `Total Ventas = SUM(Ventas[Monto])` | `Monto con IVA = Ventas[Monto] * 1.19` |

**Regla práctica:** si necesitas un valor que cambie según lo que el usuario filtra en el tablero, usa una **medida**. Si necesitas un dato nuevo para cada fila de la tabla, usa una **columna calculada**.

---

## English requirement

**Problem 3.**

An *index* speeds up a query because it stores the values of a column in a sorted structure with pointers to the corresponding rows, so the database can jump directly to the matching records instead of scanning the entire table. A *measure* in Power BI is a calculation written in DAX that is evaluated on the fly according to the current filters, which makes it ideal for aggregations such as totals, percentages, or comparisons between periods.

---

## Conclusiones

- **Series de tiempo:** convertir la fecha en índice permite usar `resample` y `rolling`. La media móvil de 12 meses revela la tendencia (creciente en el ejemplo) y el promedio por mes después de quitar la tendencia revela la estacionalidad.
- **Visualización:** el gráfico debe elegirse según la pregunta (líneas para evolución, barras para comparar, torta para pocas proporciones). Errores como el eje Y truncado o la torta saturada distorsionan la lectura y se corrigen con un eje desde cero o con barras ordenadas.
- **Optimización y métricas:** evitar `SELECT *` y funciones sobre columnas en el `WHERE`, filtrar por rango y crear índices hace las consultas más eficientes sin cambiar el resultado. En DAX, las **medidas** se calculan según el contexto de filtros y las **columnas calculadas** se almacenan fila por fila.
