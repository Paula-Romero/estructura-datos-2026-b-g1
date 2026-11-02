# Análisis de una Serie Temporal de Ventas: Tendencia, Estacionalidad, Gráficos y Optimización SQL

**Autora:** Paula Ximena Romero Villegas

---

## Resumen del trabajo

Este documento resuelve, punto por punto, las 3 actividades del enunciado y la sección **Findings** (en inglés) que vale el 20 %:

| # | Actividad pedida | Qué hice | Dónde está |
|---|---|---|---|
| 1 | Serie temporal en Python con `resample`, media móvil, y descripción de tendencia y estacionalidad | Ventas diarias 2022-2024 → agregadas por mes → medias móviles de 3 y 12 meses → índice estacional | [Punto 1](#punto-1--serie-temporal-en-python) |
| 2 | 2 visualizaciones (evolución y comparación) con título que exprese la conclusión y ejes honestos | Gráfico de evolución (columnas + media móvil) y gráfico de comparación (barras) | [Punto 2](#punto-2--dos-visualizaciones) |
| 3 | Optimizar una consulta SQL o crear 2 medidas DAX, explicando la mejora o la métrica | Optimicé una consulta SQL (medida: ~90 veces más rápida) y agregué 2 medidas DAX como extra | [Punto 3](#punto-3--optimización-de-una-consulta-sql) |
| — | README "Findings" en inglés (10 oraciones) | Redactada al final | [Findings](#findings) |

> **Sobre los datos.** El enunciado permite escoger cualquier serie, y no incluye un dataset. Por eso usé **ventas diarias simuladas**
> (semilla fija `42`, así el resultado es siempre el mismo). Los datos tienen una tendencia de crecimiento, una estacionalidad anual
> y ruido aleatorio que yo definí en el script; por eso todos los resultados describen *esta* serie. Si se reemplaza el CSV por datos reales
> (columnas `date, region, orders, revenue`), el análisis funciona sin cambiar nada.

**Cómo reproducirlo:** guardar cada bloque de código con el nombre indicado en una misma carpeta y ejecutar:
`pip install pandas numpy matplotlib` → `python generate_data.py` → `python analysis.py` → `python text_charts.py` → `python benchmark_sql.py`
(los scripts crean solas las carpetas `data/` y `charts/`).

---

## Punto 1 — Serie temporal en Python

### 1.1 Qué es lo que se pide y qué serie elegí

Una **serie temporal** es una variable medida repetidamente en el tiempo. Elegí **ventas (ingresos) diarias** de 3 regiones
(Norte, Centro y Sur) entre el 1 de enero de 2022 y el 31 de diciembre de 2024: 1,096 días x 3 regiones = 3,288 filas.

### 1.2 Pasos del análisis y justificación de cada decisión

1. **Agregar por periodo con `resample`.** Sumé los ingresos de las 3 regiones por día y luego usé `daily.resample("MS").sum()`,
   que convierte la serie diaria en **mensual** (`MS` = *Month Start*). *Justificación:* con datos diarios el ruido y el efecto
   fin de semana tapan el patrón anual; con 36 meses el patrón se ve claramente.
2. **Media móvil de 3 meses** (`rolling(3).mean()`): promedia cada mes con los 2 anteriores. Suaviza el ruido pero **conserva** el ciclo estacional.
3. **Media móvil de 12 meses** (`rolling(12).mean()`): al promediar un año completo, los meses altos y bajos se compensan, así que
   **elimina la estacionalidad** y deja visible solo la **tendencia**. Por eso sus primeros 11 valores están vacíos (la ventana aún no está completa).
4. **Tendencia:** la medí con el **crecimiento interanual** (año contra año). Probé también una regresión lineal sobre los meses, pero daba
   ≈ 21.8 % por año, un valor inflado: la serie empieza en un mes bajo (enero) y termina en el pico (diciembre), y la regresión
   confunde ese efecto de calendario con crecimiento. El crecimiento interanual compara el mismo periodo con el mismo periodo, así que la estacionalidad se cancela.
5. **Estacionalidad:** calculé un **índice estacional**. Cada mes se divide entre la media móvil centrada de 12 meses (que representa la tendencia),
   y se promedia el resultado de cada mes calendario entre los 3 años. Se normaliza para que **1.00 = mes promedio**: un 1.20 significa 20 % por encima del promedio, y un 0.73, 27 % por debajo.

### 1.3 Resultados

**Tendencia: creciente.**

| Año | Ingreso total | Crecimiento vs. año anterior |
|---|---|---|
| 2022 | 1,918,636 | — |
| 2023 | 2,201,175 | +14.7 % |
| 2024 | 2,510,129 | +14.0 % |

Crecimiento interanual medio: **14.4 % por año**. La media móvil de 12 meses sube **30.8 %** entre diciembre de 2022 y diciembre de 2024.

**Estacionalidad: anual y muy marcada, se repite los 3 años.**

| Mes | Ene | Feb | Mar | Abr | May | Jun | Jul | Ago | Sep | Oct | Nov | Dic |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Índice | 0.85 | **0.73** | 0.90 | 0.91 | 0.98 | 1.00 | 1.04 | 0.98 | 0.93 | 1.04 | 1.20 | **1.44** |

El **pico es diciembre (1.44)**, seguido de noviembre (1.20); los **meses más bajos son febrero (0.73)** y enero (0.85).
Febrero también tiene menos días (28), lo que contribuye a que su valor sea el más bajo.

### 1.4 Código

**`generate_data.py`** — crea el dataset simulado (tendencia + estacionalidad anual + efecto fin de semana + ruido)

```python
"""Generates a reproducible, SIMULATED daily sales dataset (seed fixed).
Replace data/sales_daily.csv with real data and analysis.py will work unchanged
(columns: date, region, orders, revenue)."""
import os
import numpy as np, pandas as pd
os.makedirs("data", exist_ok=True)

rng = np.random.default_rng(42)
dates = pd.date_range("2022-01-01", "2024-12-31", freq="D")
regions = {"North": 1.3, "Center": 1.0, "South": 0.7}   # relative size of each region

# Yearly seasonality: low in Q1, mild bump mid-year, strong peak Nov-Dec
month_effect = {1: .85, 2: .80, 3: .90, 4: .95, 5: 1.00, 6: 1.05,
                7: 1.05, 8: 1.00, 9: .98, 10: 1.05, 11: 1.25, 12: 1.45}
rows = []
t = np.arange(len(dates))
for reg, size in regions.items():
    trend = 1 + 0.00045 * t                               # slow growth
    season = dates.month.map(month_effect).values
    weekly = np.where(dates.dayofweek >= 5, 1.15, 1.0)    # weekends sell more
    noise = rng.normal(1, 0.08, len(dates))
    revenue = 1500 * size * trend * season * weekly * noise
    orders = np.maximum((revenue / rng.normal(42, 3, len(dates))).round(), 1)
    rows.append(pd.DataFrame({"date": dates, "region": reg,
                              "orders": orders.astype(int),
                              "revenue": revenue.round(2)}))
pd.concat(rows).to_csv("data/sales_daily.csv", index=False)
print("rows:", sum(len(r) for r in rows))
```

**`analysis.py`** — hace el `resample`, las medias móviles, la tendencia, la estacionalidad y los 2 gráficos del Punto 2

```python
"""Time-series analysis: resample -> moving average -> trend & seasonality -> 2 charts."""
import os
import numpy as np, pandas as pd
os.makedirs("charts", exist_ok=True)
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
import matplotlib.ticker as mt

# ---------- 1. Load, resample, moving average ----------
df = pd.read_csv("data/sales_daily.csv", parse_dates=["date"])
daily = df.groupby("date")["revenue"].sum()                 # total revenue per day
monthly = daily.resample("MS").sum().to_frame("revenue")    # <- resample by period (month)
monthly["ma3"] = monthly["revenue"].rolling(3).mean()       # 3-month moving average
monthly["ma12"] = monthly["revenue"].rolling(12).mean()     # 12-month MA (removes seasonality)
monthly.to_csv("data/monthly_revenue.csv")

# Trend: OLS line on the monthly series
x = np.arange(len(monthly))
slope, intercept = np.polyfit(x, monthly["revenue"].values, 1)
ma12 = monthly["ma12"].dropna()
ma12_change = (ma12.iloc[-1] / ma12.iloc[0] - 1) * 100

# Seasonality: detrend by ratio to 12M centered MA, average by calendar month
centered = monthly["revenue"].rolling(12, center=True).mean().rolling(2).mean().shift(-1)
ratio = (monthly["revenue"] / centered).dropna()
seasonal_idx = ratio.groupby(ratio.index.month).mean()
seasonal_idx = seasonal_idx / seasonal_idx.mean()           # normalise: 1.00 = average month
peak_m, low_m = seasonal_idx.idxmax(), seasonal_idx.idxmin()
mname = lambda m: pd.Timestamp(2000, m, 1).strftime("%B")

# Year-over-year totals
yearly = monthly["revenue"].groupby(monthly.index.year).sum()
yoy = yearly.pct_change().dropna() * 100
growth_pct_per_year = yoy.mean()   # YoY growth: robust to seasonality (OLS on raw months is biased by where the series starts/ends)

print(f"OLS slope: {slope:,.0f}/month (biased by seasonal start/end) | mean YoY growth {growth_pct_per_year:.1f}%/yr")
print(f"12M MA change first->last: {ma12_change:.1f}%")
print("Seasonal index:\n", seasonal_idx.round(2).to_string())
print(f"Peak {mname(peak_m)} {seasonal_idx[peak_m]:.2f} | Low {mname(low_m)} {seasonal_idx[low_m]:.2f}")
print("Yearly totals:\n", yearly.round(0).to_string()); print("YoY %:\n", yoy.round(1).to_string())

# ---------- 2. Charts ----------
plt.rcParams.update({"font.size": 10, "axes.spines.top": False, "axes.spines.right": False})
fmt_k = mt.FuncFormatter(lambda v, _: f"{v/1000:,.0f}k")

# Chart 1 - EVOLUTION (line). Y axis starts at 0 so growth is not exaggerated.
fig, ax = plt.subplots(figsize=(10, 5))
ax.plot(monthly.index, monthly["revenue"], color="#9ecae1", lw=1.5, marker="o", ms=3, label="Monthly revenue")
ax.plot(monthly.index, monthly["ma3"], color="#3182bd", lw=2, label="3-month moving average")
ax.plot(monthly.index, monthly["ma12"], color="#08306b", lw=2.5, label="12-month moving average")
ax.set_ylim(0, monthly["revenue"].max() * 1.1)
ax.yaxis.set_major_formatter(fmt_k)
ax.set_ylabel("Revenue per month (thousands, currency units)")
ax.set_xlabel("Month")
ax.set_title(f"Revenue grows ~{growth_pct_per_year:.0f}% per year, with a repeating peak every {mname(peak_m)}",
             loc="left", fontweight="bold")
ax.grid(axis="y", alpha=.3); ax.legend(frameon=False, loc="lower right")
fig.text(0.01, 0.01, "Source: simulated daily sales (seed 42), summed by month. Y axis starts at 0.", fontsize=8, color="gray")
fig.tight_layout(rect=(0, 0.03, 1, 1)); fig.savefig("charts/01_evolution.png", dpi=110); plt.close(fig)

# Chart 2 - COMPARISON (bars): seasonal index by calendar month. Bars start at 0; baseline = 1.00
fig, ax = plt.subplots(figsize=(10, 5))
colors = ["#08306b" if m == peak_m else "#d95f0e" if m == low_m else "#9ecae1" for m in seasonal_idx.index]
bars = ax.bar([mname(m)[:3] for m in seasonal_idx.index], seasonal_idx.values, color=colors)
ax.axhline(1, color="black", lw=1, ls="--"); ax.text(-0.45, 1.02, "average month = 1.00", ha="left", fontsize=8)
ax.bar_label(bars, labels=[f"{v:.2f}" for v in seasonal_idx.values], padding=2, fontsize=9)
ax.set_ylim(0, seasonal_idx.max() * 1.15)
ax.set_ylabel("Seasonal index (1.00 = average month, trend removed)")
ax.set_xlabel("Calendar month")
ax.set_title(f"{mname(peak_m)} sells {(seasonal_idx[peak_m]/seasonal_idx[low_m]-1)*100:.0f}% more than {mname(low_m)} once the trend is removed",
             loc="left", fontweight="bold")
ax.grid(axis="y", alpha=.3)
fig.text(0.01, 0.01, "Index = monthly revenue / centered 12-month average, averaged across years. Y axis starts at 0.", fontsize=8, color="gray")
fig.tight_layout(rect=(0, 0.03, 1, 1)); fig.savefig("charts/02_seasonality_comparison.png", dpi=110); plt.close(fig)
```

**Salida real de `analysis.py`**

```text
OLS slope: 2,901/month (biased by seasonal start/end) | mean YoY growth 14.4%/yr
12M MA change first->last: 30.8%
Seasonal index:
 date
1     0.85
2     0.73
3     0.90
4     0.91
5     0.98
6     1.00
7     1.04
8     0.98
9     0.93
10    1.04
11    1.20
12    1.44
Peak December 1.44 | Low February 0.73
Yearly totals:
 date
2022    1918636.0
2023    2201175.0
2024    2510129.0
YoY %:
 date
2023    14.7
2024    14.0
```

---

## Punto 2 — Dos visualizaciones

El enunciado pide una de **evolución** y una de **comparación**, con un **título que exprese la conclusión** (no solo "Ventas por mes")
y **ejes honestos**. Los gráficos están dibujados **con texto** dentro de este mismo README para que se vean en cualquier visor
(el script `text_charts.py`, incluido abajo, los calcula directamente de los datos). Además, `analysis.py` genera las versiones en imagen
(`charts/01_evolution.png` y `charts/02_seasonality_comparison.png`) con matplotlib, con los mismos datos y los mismos títulos.

### Gráfico 1 — Evolución (columnas en el tiempo)

**Título (la conclusión): "Revenue grows ~14% per year, with a repeating peak every December"**

```text
Ingreso mensual (miles de unidades monetarias)       Columnas █ = ingreso de cada mes | ━ = media móvil de 12 meses
 320k ┤                                                                        
      │                                                                      ▅ 
 280k ┤                                                                      █ 
      │                                              ▇                       █ 
 240k ┤                      ▁                       █                     █ █ 
      │                      █                     ▂ █                     █ █ 
 200k ┤                    ▁ █                     █ █           ▂ ▇ ▃ ━ ━ ━ ━ 
      │                    █ █             ▅ ▁   ▆ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ █ █ █ █ 
 160k ┤            ▂     ▅ █ █ ━ ━ ━ ━ ━ ━ ━ ━ ━ ━ █ █ ▂   █ █ █ █ █ █ █ █ █ █ 
      │        ▆ ▇ █ ▇ ▄ █ █ ━ ▂   ▆ █ █ █ █ █ █ █ █ █ █ ▂ █ █ █ █ █ █ █ █ █ █ 
 120k ┤▂   ▅ ▇ █ █ █ █ █ █ █ █ █ ▁ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ 
      │█ ▂ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ 
  80k ┤█ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ 
      │█ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ 
  40k ┤█ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ 
      │█ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ 
   0k ┤█ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ 
      └────────────────────────────────────────────────────────────────────────
       Jan-22      Jul-22      Jan-23      Jul-23      Jan-24      Jul-24
                                    Eje X: mes (2022-2024)  |  Eje Y: ingreso por mes, empieza en 0k
```

- **Qué muestra:** el ingreso de cada mes (columnas) y la media móvil de 12 meses (línea `━`).
- **Cómo se lee:** las columnas suben mucho cada fin de año (los picos de diciembre: 2022, 2023 y 2024, cada uno más alto que el anterior) y bajan en enero-febrero;
  eso es la **estacionalidad**. La línea `━` sube de forma suave y constante de ~160k a ~210k: esa es la **tendencia**, sin el efecto de los meses.
- En la imagen de matplotlib se añade además la media móvil de 3 meses.

### Gráfico 2 — Comparación (barras horizontales entre meses)

**Título (la conclusión): "December sells 98% more than February once the trend is removed"** (1.44 / 0.73 = 1.98)

```text
Índice estacional por mes (1.00 = mes promedio, ya sin la tendencia)      ┊ = nivel 1.00 (mes promedio)
Jan |███████████████████████    ┊              0.85
Feb |███████████████████        ┊              0.73  <-- lowest
Mar |████████████████████████   ┊              0.90
Apr |████████████████████████   ┊              0.91
May |██████████████████████████ ┊              0.98
Jun |███████████████████████████               1.00
Jul |████████████████████████████              1.04
Aug |██████████████████████████ ┊              0.98
Sep |█████████████████████████  ┊              0.93
Oct |████████████████████████████              1.04
Nov |████████████████████████████████          1.20
Dec |██████████████████████████████████████    1.44  <-- peak
    ++-------+-------+-------+-------+-------+
     0.0     0.3     0.6     0.9     1.2     1.5
                    Eje X: índice estacional, empieza en 0  |  Eje Y: mes calendario
```

- **Qué muestra:** cuánto vende cada mes calendario respecto a un mes promedio, una vez eliminada la tendencia.
- **Cómo se lee:** una barra que pasa la marca `┊` (1.00) es un mes fuerte; una que no llega es un mes débil.
  Diciembre (1.44) es el máximo y febrero (0.73) el mínimo; noviembre (1.20) también es fuerte.

**`text_charts.py`** — genera los dos gráficos de texto a partir de `data/monthly_revenue.csv` (lo crea `analysis.py`)

```python
"""Text version of the 2 charts (renders in any Markdown viewer). Reads data/monthly_revenue.csv."""
import pandas as pd

m = pd.read_csv("data/monthly_revenue.csv", index_col=0, parse_dates=True)

# ---------- Chart 1: evolution (columns = monthly revenue, ━ = 12-month moving average) ----------
rows, step, blocks = 17, 20_000, " ▁▂▃▄▅▆▇█"           # y axis: 0 to 340k, starts at 0
def fill(v, r):
    lo = r * step
    if v >= lo + step: return "█"
    if v <= lo: return " "
    return blocks[int((v - lo) / step * 8)]
lines = []
for r in range(rows - 1, -1, -1):
    row = ""
    for _, x in m.iterrows():
        ch = fill(x["revenue"], r)
        if pd.notna(x["ma12"]) and int(x["ma12"] // step) == r: ch = "━"
        row += ch + " "
    lines.append((f"{r*step//1000:>4}k ┤" if r % 2 == 0 else "      │") + row)
lines.append("      └" + "─" * (len(m) * 2))
axis = [" "] * (len(m) * 2)
for i, d in enumerate(m.index):
    if d.month in (1, 7):
        for k, c in enumerate(d.strftime("%b-%y")): axis[i * 2 + k] = c
lines.append("       " + "".join(axis).rstrip())
print("CHART 1\n" + "\n".join(lines))

# ---------- Chart 2: comparison (horizontal bars = seasonal index, 1.00 = average month) ----------
centered = m["revenue"].rolling(12, center=True).mean().rolling(2).mean().shift(-1)
ratio = (m["revenue"] / centered).dropna()
idx = ratio.groupby(ratio.index.month).mean()
idx = idx / idx.mean()
names = ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"]
scale, ref = 40 / 1.5, round(40 / 1.5)                  # 40 characters = 1.5; x axis starts at 0
out = []
for mth, nm in zip(range(1, 13), names):
    n = round(idx[mth] * scale)
    bar = list("█" * n + " " * (41 - n))
    if n < ref: bar[ref] = "┊"                          # reference mark at 1.00
    tag = "  <-- peak" if idx[mth] == idx.max() else "  <-- lowest" if idx[mth] == idx.min() else ""
    out.append(f"{nm} |{''.join(bar)} {idx[mth]:.2f}{tag}")
out.append("    +" + "".join("+" if i % 8 == 0 else "-" for i in range(41)))
out.append("     " + "0.0".ljust(8) + "0.3".ljust(8) + "0.6".ljust(8) + "0.9".ljust(8) + "1.2".ljust(8) + "1.5")
print("\nCHART 2\n" + "\n".join(out))
```

### Por qué los ejes son honestos

- **Ambos ejes Y empiezan en 0.** Un eje truncado (por ejemplo, que empiece en 100k) haría que un crecimiento moderado pareciera enorme, y en el de barras exageraría la diferencia entre meses.
- **Cada eje tiene nombre y unidades** (miles de unidades monetarias; índice estacional con su significado).
- El gráfico de barras tiene una **línea de referencia en 1.00** para que se entienda qué es "normal".
- Cada gráfico indica qué representa cada símbolo y el método (el índice se calcula como ingreso del mes / media móvil centrada de 12 meses).
- No se usó color para engañar: solo se destacan los dos meses extremos que menciona el título.

---

## Punto 3 — Optimización de una consulta SQL

### 3.1 La consulta y el objetivo

Tabla `sales(sale_id, sale_date, region, orders, revenue)`. Objetivo: **ingreso mensual de 2024 con su media móvil de 3 meses**
(el mismo indicador que calculé en Python). Probé las dos versiones en SQLite con **1,000,000 de filas** para medir la mejora de verdad.

### 3.2 Qué estaba mal y cómo lo arreglé

| Problema en la consulta original | Por qué es lento | Corrección |
|---|---|---|
| `WHERE strftime('%Y', sale_date) = '2024'` | Aplica una función a la columna: es **no sargable**. El motor no puede usar un índice y debe calcular la función en cada fila (escaneo completo). | Usar un **rango** (`sale_date >= '2023-10-01' AND sale_date < '2025-01-01'`) y crear un **índice cubriente** `(sale_date, revenue)`. |
| Media móvil con una **subconsulta correlacionada** | Para cada mes vuelve a agrupar toda la tabla (una nueva pasada por mes). | Agregar **una sola vez** en una CTE y calcular la media con una **función ventana**: `AVG(revenue) OVER (ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)`. |
| Enero y febrero necesitan los meses previos | Si se filtra solo 2024, se pierde su historia. | La CTE lee desde octubre de 2023 y el filtro de 2024 se aplica en una consulta **externa**. |

> **Un error que encontré al verificar:** mi primera versión filtraba `month >= '2024-01'` en la misma consulta de la ventana. En SQL, el `WHERE`
> se ejecuta **antes** de las funciones ventana, así que Enero y Febrero perdían sus meses previos y su media salía mal.
> Lo detecté porque el script comparaba los resultados de ambas versiones y no coincidían; lo corregí con la consulta externa. Ahora son **idénticos**.

### 3.3 Código SQL (antes y después)

```sql
-- =====================================================================
-- Table: sales(sale_id, sale_date TEXT 'YYYY-MM-DD', region, orders, revenue)
-- Goal: monthly revenue for 2024 + 3-month moving average (same indicator as analysis.py)
-- =====================================================================

-- ---------- BEFORE (slow) ----------
-- Problems:
--  (1) strftime()/YEAR() on the column in WHERE is NON-SARGABLE: the engine cannot use an
--      index on sale_date and must evaluate the function on every row (full table scan).
--  (2) The moving average uses a correlated subquery: it re-scans the table once per month.
SELECT m.month,
       m.revenue,
       (SELECT AVG(m2.revenue)
          FROM (SELECT strftime('%Y-%m', sale_date) AS month, SUM(revenue) AS revenue
                  FROM sales GROUP BY 1) m2
         WHERE m2.month <= m.month
           AND m2.month >  strftime('%Y-%m', date(m.month || '-01', '-3 months'))) AS ma3
FROM (SELECT strftime('%Y-%m', sale_date) AS month, SUM(revenue) AS revenue
        FROM sales
       WHERE strftime('%Y', sale_date) = '2024'
       GROUP BY 1) m
ORDER BY m.month;

-- ---------- AFTER (optimized) ----------
-- Fixes:
--  (1) Index on sale_date + range predicate (sargable) -> index range SEARCH instead of SCAN.
--  (2) Aggregate ONCE in a CTE, then use a WINDOW function for the moving average (single pass).
--  (3) The CTE reads from 2023-10-01 so Jan/Feb 2024 have their 2 previous months available.
--      IMPORTANT: WHERE runs BEFORE window functions, so the 2024 filter must be applied in an
--      OUTER query, after the window is computed (otherwise Jan/Feb would lose their history).
CREATE INDEX IF NOT EXISTS idx_sales_date_rev ON sales(sale_date, revenue);   -- covering index

WITH monthly AS (
    SELECT substr(sale_date, 1, 7) AS month,
           SUM(revenue)            AS revenue
    FROM   sales
    WHERE  sale_date >= '2023-10-01' AND sale_date < '2025-01-01'
    GROUP  BY substr(sale_date, 1, 7)
),
with_ma AS (
    SELECT month,
           revenue,
           AVG(revenue) OVER (ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS ma3
    FROM   monthly
)
SELECT month, revenue, ma3
FROM   with_ma
WHERE  month >= '2024-01'
ORDER  BY month;
```

### 3.4 Métrica: resultado medido

| | Tiempo | Plan de ejecución (`EXPLAIN QUERY PLAN`) |
|---|---|---|
| **Antes** | **13.70 s** | `SCAN sales` (dos veces: escaneo completo de la tabla) |
| **Después** | **0.15 s** | `SEARCH sales USING COVERING INDEX idx_sales_date_rev` (búsqueda por rango en el índice) |

**La consulta optimizada es ≈ 93 veces más rápida y devuelve exactamente los mismos 12 resultados.** Los tiempos exactos dependen de la computadora
(en una ejecución anterior fueron 11.4 s y 0.2 s, ≈ 55x); lo importante y estable es el cambio del plan: de `SCAN` (leer todo) a `SEARCH USING INDEX` (leer solo el rango necesario).

**`benchmark_sql.py`** — crea la tabla de 1,000,000 de filas, ejecuta ambas versiones y compara tiempos, planes y resultados

```python
"""Runs BEFORE/AFTER queries on SQLite (1,000,000 rows); checks equal results, plans and timings."""
import sqlite3, time, numpy as np, pandas as pd, re
rng = np.random.default_rng(1)
n = 1_000_000
days = pd.date_range("2022-01-01", "2024-12-31")
d = days[rng.integers(0, len(days), n)].strftime("%Y-%m-%d")
con = sqlite3.connect(":memory:")
con.execute("CREATE TABLE sales(sale_id INTEGER PRIMARY KEY, sale_date TEXT, region TEXT, orders INT, revenue REAL)")
con.executemany("INSERT INTO sales(sale_date,region,orders,revenue) VALUES (?,?,?,?)",
    zip(d, rng.choice(["North","Center","South"], n), rng.integers(1, 9, n), rng.uniform(10, 400, n).round(2)))
BEFORE = """
SELECT m.month,
       m.revenue,
       (SELECT AVG(m2.revenue)
          FROM (SELECT strftime('%Y-%m', sale_date) AS month, SUM(revenue) AS revenue
                  FROM sales GROUP BY 1) m2
         WHERE m2.month <= m.month
           AND m2.month >  strftime('%Y-%m', date(m.month || '-01', '-3 months'))) AS ma3
FROM (SELECT strftime('%Y-%m', sale_date) AS month, SUM(revenue) AS revenue
        FROM sales
       WHERE strftime('%Y', sale_date) = '2024'
       GROUP BY 1) m
ORDER BY m.month;
"""

CREATE_IDX = """CREATE INDEX IF NOT EXISTS idx_sales_date_rev ON sales(sale_date, revenue);"""

AFTER = """
WITH monthly AS (
    SELECT substr(sale_date, 1, 7) AS month,
           SUM(revenue)            AS revenue
    FROM   sales
    WHERE  sale_date >= '2023-10-01' AND sale_date < '2025-01-01'
    GROUP  BY substr(sale_date, 1, 7)
),
with_ma AS (
    SELECT month,
           revenue,
           AVG(revenue) OVER (ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS ma3
    FROM   monthly
)
SELECT month, revenue, ma3
FROM   with_ma
WHERE  month >= '2024-01'
ORDER  BY month;
"""

before, create_idx, after = BEFORE, CREATE_IDX, AFTER

def run(q):
    t = time.perf_counter(); rows = con.execute(q).fetchall(); return rows, time.perf_counter() - t
def plan(q): return [r[3] for r in con.execute("EXPLAIN QUERY PLAN " + q)]

rb, tb = run(before); pb = plan(before)
con.execute(create_idx)
ra, ta = run(after);  pa = plan(after)
same = all(a[0]==b[0] and abs(a[1]-b[1])<1e-6 and abs(a[2]-b[2])<1e-6 for a, b in zip(rb, ra)) and len(rb)==len(ra)
print(f"rows in table: {n:,}")
print(f"BEFORE: {tb:.3f}s  plan: {pb}")
print(f"AFTER : {ta:.3f}s  plan: {pa}")
print(f"speed-up: {tb/ta:.1f}x | identical results: {same} | result rows: {len(ra)}")
```

**Salida real del benchmark**

```text
rows in table: 1,000,000
BEFORE: 13.703s  plan: ['CO-ROUTINE m', 'SCAN sales', 'USE TEMP B-TREE FOR GROUP BY', 'SCAN m', 'CORRELATED SCALAR SUBQUERY 2', 'CO-ROUTINE m2', 'SCAN sales', 'USE TEMP B-TREE FOR GROUP BY', 'SCAN m2', 'USE TEMP B-TREE FOR ORDER BY']
AFTER : 0.148s  plan: ['CO-ROUTINE with_ma', 'CO-ROUTINE (subquery-4)', 'CO-ROUTINE monthly', 'SEARCH sales USING COVERING INDEX idx_sales_date_rev (sale_date>? AND sale_date<?)', 'USE TEMP B-TREE FOR GROUP BY', 'SCAN monthly', 'USE TEMP B-TREE FOR ORDER BY', 'SCAN (subquery-4)', 'SCAN with_ma', 'USE TEMP B-TREE FOR ORDER BY']
speed-up: 92.7x | identical results: True | result rows: 12
```

### 3.5 Extra: 2 medidas DAX

El enunciado dice "SQL **o** 2 medidas DAX"; con el SQL ya se cumple, pero agrego las dos medidas DAX equivalentes a los indicadores del Punto 1.
Suponen un modelo con la tabla `Sales` y una tabla `Calendar` marcada como tabla de fechas.
*Nota: no pude ejecutarlas aquí (requieren Power BI), por lo que no están probadas.*

- **`Revenue 3M MA`**: media móvil de 3 meses (equivale a `rolling(3).mean()` de Python). Usa `DATESINPERIOD`, que mira los 3 últimos meses en el contexto de cada mes.
- **`Revenue YoY %`**: crecimiento contra el mismo periodo del año anterior (la métrica con la que medí la tendencia: ~14 %). Usa `SAMEPERIODLASTYEAR`, y por eso la estacionalidad se cancela.
- Ambas usan `DIVIDE` (evita errores de división entre cero) y variables (`VAR`), que evalúan cada expresión una sola vez.

```dax
// Model: fact table Sales[Date, Region, Orders, Revenue] + marked Date table 'Calendar'[Date]
// (relationship Calendar[Date] 1 -> * Sales[Date]; Calendar marked as date table)

// ---------- Measure 1: 3-month moving average of monthly revenue ----------
Revenue 3M MA =
VAR _Last3Months = DATESINPERIOD ( 'Calendar'[Date], MAX ( 'Calendar'[Date] ), -3, MONTH )
RETURN
    DIVIDE (
        CALCULATE ( SUM ( Sales[Revenue] ), _Last3Months ),
        3
    )
// Why: DATESINPERIOD is evaluated in the filter context of each month -> same indicator as
// the Python rolling(3).mean(). DIVIDE avoids a division error and SUM stays in the storage
// engine (fast); no row-by-row iterator (SUMX) is needed.

// ---------- Measure 2: Year-over-Year growth % ----------
Revenue YoY % =
VAR _Current = SUM ( Sales[Revenue] )
VAR _PrevYear = CALCULATE ( SUM ( Sales[Revenue] ), SAMEPERIODLASTYEAR ( 'Calendar'[Date] ) )
RETURN
    DIVIDE ( _Current - _PrevYear, _PrevYear )   // BLANK when there is no previous year
// Why: compares the same period with the previous year, so seasonality cancels out
// (December vs December). This is the metric used in the README to quantify the trend (~14%/yr).
// Variables evaluate each expression once. Format as percentage in the model.
```

---

## Findings

The monthly revenue series, built by resampling daily sales with resample("MS"), shows a clear upward trend over the three years analysed. Yearly revenue grew from 1.92 million in 2022 to 2.20 million in 2023 and 2.51 million in 2024, which is an average year-over-year growth of about 14.4%. The 12-month moving average, which removes the seasonal cycle, rises 30.8% between December 2022 and December 2024, so the growth is steady and not caused by a single month. The series also shows a strong and repeating yearly seasonality, with the highest revenue in November and December and the lowest in January and February. The first chart, titled "Revenue grows ~14% per year, with a repeating peak every December", shows the revenue of each month as a column together with the 12-month moving average, on an axis that starts at zero. In that chart, the columns jump every December and fall in January and February, while the 12-month average is a smooth rising line that isolates the trend. The second chart, titled "December sells 98% more than February once the trend is removed", compares the seasonal index of each calendar month, where 1.00 represents an average month. In that chart, December scores 1.44, November 1.20 and February 0.73, and because the index removes the trend, this gap is a pure calendar effect and not the result of growth over time. A practical consequence is that comparing one month with the previous month is misleading in this series, so year-over-year comparisons or the 12-month moving average should be used instead. Finally, the dataset is simulated, so these findings describe this specific dataset and should be validated again with real data.
