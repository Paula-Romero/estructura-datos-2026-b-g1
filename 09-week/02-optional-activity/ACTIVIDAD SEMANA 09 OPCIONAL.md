# Calidad de datos: limpieza y evidencia antes/después

**Caso:** una cadena de tiendas de ropa con sedes en Neiva, Bogotá, Medellín y Cali. En esta actividad se trabaja con los **clientes de la tienda en línea**, una de las fuentes del caso, porque suele ser la que llega con más problemas de calidad (formularios escritos a mano, registros repetidos, formatos mezclados).

| | |
|---|---|
| **Estudiante** | `FULL_NAME` |
| **Usuario GitHub** | `GITHUB_USER` |
| **Herramientas** | Python 3, pandas |

## Objetivos

- Detectar y corregir problemas de calidad.
- Documentar el antes/después.

## Índice

Este README responde los **3 puntos** del enunciado, en orden:

| Punto | Qué pide el enunciado | Dónde está |
|---|---|---|
| **Punto 1** | Tomar un dataset con problemas y, en pandas: contar nulos, eliminar/imputar faltantes, quitar duplicados, corregir tipos y normalizar formatos de texto | [Ir al Punto 1](#punto-1-dataset-con-problemas-y-limpieza-en-pandas) |
| **Punto 2** | Reportar antes/después (nº de filas, nulos, duplicados) | [Ir al Punto 2](#punto-2-reporte-antesdespués) |
| **Punto 3** | Comentar 2 dimensiones de calidad que mejoraste | [Ir al Punto 3](#punto-3-dos-dimensiones-de-calidad-que-mejoré) |
| **Requisito** | Script de limpieza en pandas + evidencia antes/después | Script en el Punto 1 (sección 1.4); evidencia en el Punto 2 |

---

## Punto 1: Dataset con problemas y limpieza en pandas

> **Enunciado:** toma un dataset con problemas (o ensúcialo tú) y en Python (pandas): cuenta nulos, elimina/imputa faltantes, quita duplicados, corrige tipos y normaliza formatos de texto.

### 1.1 El dataset

Se usa **`clientes_raw.csv`**: **216 filas y 8 columnas**, con los clientes registrados en la tienda en línea. Es un dataset simulado que **ensucié yo** con `generar_dataset.py` (semilla fija, siempre da el mismo resultado), con problemas típicos de datos reales.

| Columna | Qué guarda |
|---|---|
| `cliente_id` | Identificador del cliente |
| `nombre` | Nombre completo |
| `correo` | Correo electrónico |
| `telefono` | Celular |
| `ciudad` | Ciudad del cliente |
| `fecha_registro` | Fecha en que se registró |
| `edad` | Edad en años |
| `canal_registro` | Web, App o Tienda |

**Primeras 5 filas del dataset original:**

|   cliente_id | nombre          | correo                         | telefono         | ciudad   | fecha_registro   | edad    | canal_registro   |
|-------------:|:----------------|:-------------------------------|:-----------------|:---------|:-----------------|:--------|:-----------------|
|          139 | Juliana Cortés  | JULIANA.CORTES139@HOTMAIL.COM  | +57 366-321-4572 | Medellin | 2026-07-08       | NaN     | Tienda           |
|           66 | valentina silva | @gmail.com                     | NaN              | BOGOTA   | 2026-08-19       | 39 años | Tienda           |
|           96 | LAURA DÍAZ      | laura.diaz96@gmail.com         | 3925665965       | neiva    | 2025-06-01       | 42      | Tienda           |
|           23 | Sebastián Díaz  | sebastian.diaz23@hotmail.com   | 3138924717       | Neiva    | 20/06/2026       | 25      | Tienda           |
|           64 | Camila  Gómez   | CAMILA.GOMEZ64@CORHUILA.EDU.CO | NaN              | NEIVA    | 2025-09-18       | 34      | APP              |

<details>
<summary><b>Ver el código que genera el dataset (generar_dataset.py)</b></summary>

```python
"""Genera clientes_raw.csv: clientes de la tienda en línea, con problemas de calidad a propósito."""
import unicodedata
import numpy as np
import pandas as pd

rng = np.random.default_rng(2026)
N = 200

nombres = ["María", "José", "Laura", "Carlos", "Ana", "Juan", "Camila", "Andrés", "Valentina", "Santiago",
           "Daniela", "Felipe", "Sofía", "Miguel", "Paula", "David", "Natalia", "Sebastián", "Juliana", "Jorge"]
apellidos = ["Pérez", "Gómez", "Rodríguez", "Martínez", "López", "García", "Hernández", "Ramírez", "Torres", "Vargas",
             "Castro", "Rojas", "Moreno", "Ortiz", "Silva", "Cortés", "Muñoz", "Díaz", "Suárez", "Quintero"]
ciudades = ["Neiva", "Bogotá", "Medellín", "Cali"]
canales = ["Web", "App", "Tienda"]
dominios = ["gmail.com", "hotmail.com", "outlook.com", "corhuila.edu.co"]

def ascii_(t):
    return unicodedata.normalize("NFKD", t).encode("ascii", "ignore").decode().lower()

# ---------- 1) Datos base (limpios) ----------
rows = []
for i in range(1, N + 1):
    n, a = str(rng.choice(nombres)), str(rng.choice(apellidos))
    rows.append({
        "cliente_id": i,
        "nombre": f"{n} {a}",
        "correo": f"{ascii_(n)}.{ascii_(a)}{i}@{rng.choice(dominios)}",
        "telefono": "3" + "".join(map(str, rng.integers(0, 10, 9))),
        "ciudad": str(rng.choice(ciudades)),
        "fecha_registro": pd.Timestamp("2025-01-01") + pd.Timedelta(days=int(rng.integers(0, 630))),
        "edad": int(rng.integers(18, 71)),
        "canal_registro": str(rng.choice(canales)),
    })
base = pd.DataFrame(rows)

# ---------- 2) Clientes repetidos (mismo correo, otro cliente_id) ----------
rep = base.sample(6, random_state=3).copy()
rep["cliente_id"] = range(N + 1, N + 7)
rep["fecha_registro"] = rep["fecha_registro"] + pd.to_timedelta(rng.integers(30, 200, 6), unit="D")
df = pd.concat([base, rep], ignore_index=True)
protegidos = set(df.index[df["correo"].isin(rep["correo"])])   # no dañar el correo de los repetidos

# ---------- 3) Ensuciar formatos ----------
def s_nombre(x):
    r = rng.random()
    if r < 0.30: return x.upper()
    if r < 0.45: return x.lower()
    if r < 0.55: return "  " + x.replace(" ", "  ") + " "
    return x
def s_correo(x):
    r = rng.random()
    if r < 0.15: return x.upper()
    if r < 0.25: return x + " "
    return x
def s_tel(x):
    r = rng.random()
    if r < 0.25: return f"{x[:3]} {x[3:6]} {x[6:]}"
    if r < 0.40: return f"+57 {x[:3]}-{x[3:6]}-{x[6:]}"
    if r < 0.50: return f"({x[:3]}) {x[3:]}"
    return x
VAR = {"Neiva": ["Neiva", "neiva", "NEIVA "], "Bogotá": ["Bogotá", "Bogota", "BOGOTA", "bogotá", "Bogotá D.C."],
       "Medellín": ["Medellín", "Medellin", "MEDELLIN ", "medellín"], "Cali": ["Cali", "cali", "CALI", "Cali "]}
def s_ciudad(c): return str(rng.choice(VAR[c]))
def s_fecha(d): return d.strftime("%Y-%m-%d") if rng.random() < 0.65 else d.strftime("%d/%m/%Y")
def s_edad(e):
    r = rng.random()
    return str(e) if r < 0.80 else (f"{e} años" if r < 0.92 else f"{e}.0")
def s_canal(c):
    r = rng.random()
    return c if r < 0.6 else (c.lower() if r < 0.8 else (c.upper() if r < 0.9 else c + " "))

for col, f in [("nombre", s_nombre), ("correo", s_correo), ("telefono", s_tel), ("ciudad", s_ciudad),
               ("fecha_registro", s_fecha), ("edad", s_edad), ("canal_registro", s_canal)]:
    df[col] = df[col].map(f)

# ---------- 4) Nulos y valores inválidos ----------
libres = [i for i in df.index if i not in protegidos]
def pick(k, pool):
    return list(rng.choice(pool, k, replace=False))
for col, k in [("nombre", 4), ("telefono", 22), ("ciudad", 10), ("edad", 16), ("canal_registro", 6)]:
    df.loc[pick(k, list(df.index)), col] = np.nan
i_null = pick(8, libres); df.loc[i_null, "correo"] = np.nan
i_inv = pick(6, [i for i in libres if i not in i_null])
df.loc[i_inv, "correo"] = ["correo.sin.arroba.com", "maria@", "@gmail.com", "juan perez@mail.com", "ana@@hotmail.com", "sofia.gmail.com"]
df.loc[pick(5, list(df.index)), "telefono"] = ["12345", "0000", "3101", "999", "31012"]
df.loc[pick(3, list(df.index)), "edad"] = ["-5", "150", "0"]

# ---------- 5) Duplicados exactos ----------
df = pd.concat([df, df.sample(10, random_state=5)], ignore_index=True)
df = df.sample(frac=1, random_state=7).reset_index(drop=True)
df.to_csv("clientes_raw.csv", index=False)
print("clientes_raw.csv generado:", df.shape)
```

</details>

<details>
<summary><b>Ver el dataset original completo (clientes_raw.csv, 216 filas)</b></summary>

```csv
cliente_id,nombre,correo,telefono,ciudad,fecha_registro,edad,canal_registro
139,Juliana Cortés,JULIANA.CORTES139@HOTMAIL.COM,+57 366-321-4572,Medellin,2026-07-08,,Tienda
66,valentina silva,@gmail.com,,BOGOTA,2026-08-19,39 años,Tienda
96,LAURA DÍAZ,laura.diaz96@gmail.com,3925665965,neiva,2025-06-01,42,Tienda
23,Sebastián Díaz,sebastian.diaz23@hotmail.com,3138924717,Neiva,20/06/2026,25,Tienda
64,  Camila  Gómez ,CAMILA.GOMEZ64@CORHUILA.EDU.CO,,NEIVA ,2025-09-18,34,APP
47,Camila Quintero,camila.quintero47@hotmail.com,328 330 9834,Bogotá D.C.,2026-08-30,,web
98,SEBASTIÁN SILVA,sebastian.silva98@corhuila.edu.co,3547495697,bogotá,2026-06-20,67,App
12,Santiago Rojas,santiago.rojas12@outlook.com,,medellín,2026-02-07,22,App
3,Miguel Torres,miguel.torres3@hotmail.com,999,Bogotá,2026-02-22,62,Web
83,  Juliana  Suárez ,JULIANA.SUAREZ83@CORHUILA.EDU.CO,0000,Medellín,2025-12-02,22.0,Web
25,María Vargas,maria.vargas25@outlook.com ,3556044951,CALI,2025-07-01,,APP
103,NATALIA CORTÉS,NATALIA.CORTES103@OUTLOOK.COM,+57 381-289-2217,MEDELLIN ,27/04/2025,50,Tienda
97,Daniela Gómez,daniela.gomez97@gmail.com,3519277654,NEIVA ,2025-06-21,41,App 
75,Camila Quintero,camila.quintero75@gmail.com,3291812763,,07/08/2026,42,Web
116,JOSÉ RAMÍREZ,jose.ramirez116@outlook.com,3449990516,NEIVA ,2025-02-17,52,App
163,Andrés López,andres.lopez163@corhuila.edu.co,3786893859,cali,2026-08-04,66,Tienda
158,SEBASTIÁN CASTRO,sebastian.castro158@outlook.com,338 565 5956,Bogotá,2025-06-22,25,App 
171,DANIELA SILVA,daniela.silva171@hotmail.com ,,neiva,2025-04-25,61.0,Web
86,ANDRÉS GARCÍA,,+57 347-765-5577,Bogotá D.C.,2026-06-19,19,App
110,  Valentina  Moreno ,sofia.gmail.com,(300) 5850067,,2025-05-19,,App
133,Daniela Ortiz,daniela.ortiz133@outlook.com ,3681147848,neiva,2026-03-17,54,Tienda
165,CARLOS MORENO,carlos.moreno165@corhuila.edu.co,+57 320-688-4118,Medellin,2026-03-29,60,Tienda 
81,MARÍA MUÑOZ,maria.munoz81@outlook.com,3432705205,bogotá,06/08/2026,,APP
194,Santiago López,santiago.lopez194@outlook.com ,+57 354-889-3142,BOGOTA,21/06/2025,67,App
118,Felipe Vargas,felipe.vargas118@gmail.com,12345,Bogotá D.C.,10/03/2026,28,Web
185,Jorge López,jorge.lopez185@hotmail.com,3165240988,BOGOTA,2026-06-01,32 años,Web
199,PAULA SUÁREZ,paula.suarez199@gmail.com,317 603 6388,,2025-11-30,42,App
162,juan silva,juan.silva162@hotmail.com,3203527432,Bogotá D.C.,2026-01-16,49,app
100,camila vargas,camila.vargas100@outlook.com,3279370357,BOGOTA,05/06/2025,63,Web
89,miguel garcía,miguel.garcia89@gmail.com,3283759543,Medellin,2025-02-18,69,Tienda
26,Natalia Cortés,NATALIA.CORTES26@OUTLOOK.COM,,medellín,2025-02-19,,WEB
105,JORGE RAMÍREZ,jorge.ramirez105@corhuila.edu.co,3230808810,cali,2026-02-13,48,app
61,JOSÉ LÓPEZ,jose.lopez61@gmail.com,3311948009,Cali,2025-01-09,30,APP
129,FELIPE LÓPEZ,felipe.lopez129@gmail.com,3236206977,Bogotá,2025-06-19,52.0,
102,Juliana Suárez,juliana.suarez102@corhuila.edu.co,393 358 1932,Bogotá D.C.,04/01/2025,38.0,Web
195,Daniela García,daniela.garcia195@gmail.com,,neiva,2026-05-02,52,web
79,sofía silva,sofia.silva79@hotmail.com ,3712670973,MEDELLIN ,2025-06-22,24,App
58,  Daniela  Díaz ,daniela.diaz58@corhuila.edu.co,375 280 7845,,09/08/2026,19.0,app
117,JULIANA ROJAS,juliana.rojas117@gmail.com,3483672279,BOGOTA,2025-05-11,45,Tienda
10,carlos cortés,carlos.cortes10@outlook.com,3998962119,Bogotá D.C.,15/01/2025,19 años,Web
95,  Paula  Díaz ,PAULA.DIAZ95@CORHUILA.EDU.CO,3260907748,NEIVA ,2026-07-09,37,Tienda
198,Camila Rojas,camila.rojas198@gmail.com,3043729676,Cali ,2026-03-07,66,Tienda
107,Sofía Muñoz,sofia.munoz107@gmail.com,,Cali ,2025-08-23,56 años,App
134,maría ortiz,maria.ortiz134@outlook.com ,3883501065,cali,2025-01-08,62,Tienda
126,daniela muñoz,daniela.munoz126@hotmail.com,,neiva,2026-09-01,54,Web
202,sofía gómez,sofia.gomez52@hotmail.com,3782381936,Bogotá,2026-02-15,37,Web
67,juan garcía,JUAN.GARCIA67@OUTLOOK.COM,390 748 7169,NEIVA ,20/04/2026,23,Web
167,Miguel Rojas,miguel.rojas167@gmail.com,320 948 0956,Medellin,18/05/2025,43,APP
15,Felipe Ortiz,felipe.ortiz15@corhuila.edu.co,+57 314-230-4095,BOGOTA,2025-05-26,67,APP
28,andrés garcía,andres.garcia28@hotmail.com,+57 394-571-1464,Neiva,2026-02-06,66.0,App
41,ANDRÉS MUÑOZ,andres.munoz41@outlook.com,,Bogota,2025-07-18,56 años,APP
46,felipe ortiz,FELIPE.ORTIZ46@GMAIL.COM,3137873953,medellín,2026-07-05,41,Web 
161,LAURA SUÁREZ,laura.suarez161@gmail.com,(363) 7936744,BOGOTA,2026-03-24,69,Web
32,NATALIA RODRÍGUEZ,natalia.rodriguez32@gmail.com,3259013109,Neiva,27/01/2026,64,Tienda
166,JUAN HERNÁNDEZ,juan.hernandez166@outlook.com,3981878905,NEIVA ,2025-04-05,35,Tienda
132,natalia suárez,natalia.suarez132@gmail.com,,bogotá,21/07/2025,,App
33,maría ramírez,maria.ramirez33@gmail.com,3949450985,neiva,05/10/2025,41,Tienda
14,LAURA CORTÉS,laura.cortes14@hotmail.com,3026583693,neiva,2026-02-18,49 años,App
130,DAVID GÓMEZ,david.gomez130@gmail.com,3698789171,Bogotá D.C.,2026-02-19,36,Web
144,Juliana Silva,juliana.silva144@hotmail.com,+57 361-433-8357,CALI,2025-10-13,59,Tienda
141,Juliana Ortiz,juliana.ortiz141@gmail.com,307 952 8270,Bogota,2025-11-09,48,Tienda
29,SEBASTIÁN TORRES,sebastian.torres29@hotmail.com,335 519 1538,Medellín,2025-01-10,,Web
19,Andrés Quintero,andres.quintero19@gmail.com,3201636172,Neiva,2026-07-31,37,Web
204,camila rojas,camila.rojas198@gmail.com ,,Cali,2026-04-22,66,tienda
122,PAULA SUÁREZ,PAULA.SUAREZ122@HOTMAIL.COM,3313679850,neiva,2025-12-13,51,App 
16,Jorge Ortiz,JORGE.ORTIZ16@CORHUILA.EDU.CO,3873738301,Bogota,2026-02-22,33,TIENDA
99,natalia garcía,,371 899 1035,medellín,2025-02-08,53,app
59,Valentina López,valentina.lopez59@corhuila.edu.co,313 426 6013,CALI,2025-09-12,40,Tienda
37,Camila Castro,camila.castro37@outlook.com,3734433924,neiva,2026-04-16,,App
170,ANA TORRES,ana.torres170@gmail.com,+57 328-297-0303,Bogota,08/05/2026,36,Tienda
192,Sebastián Díaz,sebastian.diaz192@gmail.com,3944080792,neiva,2025-12-21,65,Web
52,SOFÍA GÓMEZ,sofia.gomez52@hotmail.com,3782381936,BOGOTA,2025-09-06,37,Web
180,  Andrés  Ortiz ,andres.ortiz180@outlook.com ,,Cali ,2025-09-20,65,Web
159,Felipe Rodríguez,,3398141833,cali,2026-01-28,,Web
49,Carlos Torres,carlos.torres49@hotmail.com,3362295551,Medellin,2026-04-22,20,App
193,Miguel López,miguel.lopez193@hotmail.com,,Cali,2025-05-24,18 años,Tienda
146,Felipe Hernández,felipe.hernandez146@outlook.com,3110003169,Cali,2025-07-26,64.0,tienda
148,JUAN CORTÉS,juan.cortes148@gmail.com,3590528157,neiva,12/09/2025,25,App
106,David Rojas,david.rojas106@corhuila.edu.co ,+57 340-071-1618,Bogotá,31/10/2025,,App
149,María Moreno,maria.moreno149@hotmail.com,337 947 3631,,2025-08-22,36,app
38,josé rodríguez,ana@@hotmail.com,3125579743,Cali ,06/01/2025,56,app
142,SOFÍA ORTIZ,SOFIA.ORTIZ142@CORHUILA.EDU.CO,3478098567,Medellin,2026-09-12,52,Web 
85,Valentina Cortés,valentina.cortes85@gmail.com,357 320 4363,MEDELLIN ,21/02/2025,70,Web 
183,Daniela Ramírez,DANIELA.RAMIREZ183@GMAIL.COM,3844205580,Cali,03/04/2026,56.0,App
109,juan silva,JUAN.SILVA109@HOTMAIL.COM,3167818095,Neiva,2026-05-08,60,app
50,Juan Rojas,juan.rojas50@corhuila.edu.co,3306361238,Medellin,25/01/2026,55,App
121,  Laura  Cortés ,laura.cortes121@corhuila.edu.co,(349) 0706937,cali,2025-11-21,40,Web
6,,valentina.ortiz6@hotmail.com,3302174254,Bogota,2026-01-01,51,WEB
18,paula silva,paula.silva18@corhuila.edu.co,3893666654,Neiva,2025-03-06,18,
147,FELIPE GÓMEZ,felipe.gomez147@gmail.com ,366 556 3873,Medellin,2026-02-24,33,Web
181,ANDRÉS PÉREZ,correo.sin.arroba.com,3140490656,neiva,2026-09-10,56,APP
191,Valentina Pérez,valentina.perez191@hotmail.com,(302) 8167706,CALI,2025-07-18,61,App 
114,María Gómez,MARIA.GOMEZ114@GMAIL.COM,394 029 7339,Bogotá D.C.,2025-03-24,53,Web
87,MIGUEL MORENO,miguel.moreno87@hotmail.com,3900464764,Bogota,2026-04-18,44,TIENDA
63,Sofía Rojas,sofia.rojas63@gmail.com,3331314872,MEDELLIN ,2026-07-18,68 años,Web
65,LAURA QUINTERO,laura.quintero65@hotmail.com,+57 326-579-6435,CALI,2025-12-11,47,web
27,Sebastián Díaz,,383 225 2264,medellín,2025-01-14,23,web
135,carlos martínez,carlos.martinez135@corhuila.edu.co,+57 388-381-9310,Medellín,2025-05-07,18,Web
42,CARLOS DÍAZ,carlos.diaz42@hotmail.com,,medellín,25/03/2026,61,app
53,JULIANA CORTÉS,juliana.cortes53@corhuila.edu.co,3590354014,Cali,2026-01-22,33,Tienda
187,SEBASTIÁN ROJAS,sebastian.rojas187@gmail.com,3447244898,Bogota,2025-01-15,50.0,app
82,JORGE CORTÉS,jorge.cortes82@hotmail.com,+57 310-849-4584,Medellín,27/07/2026,30,Web
8,David Rojas,david.rojas8@corhuila.edu.co ,3470078573,Medellín,11/11/2025,34 años,Web
92,José Moreno,jose.moreno92@hotmail.com,3575063660,Bogotá D.C.,05/04/2025,54,app
188,Juliana Pérez,juliana.perez188@outlook.com,(335) 3070275,medellín,21/10/2025,38,
172,Carlos Suárez,carlos.suarez172@gmail.com,3389179256,bogotá,04/08/2025,37,Tienda
71,ANA DÍAZ,ana.diaz71@outlook.com,3352478474,NEIVA ,08/09/2026,64,Tienda
179,LAURA CASTRO,laura.castro179@outlook.com ,(390) 6897808,Bogotá,2025-01-20,18,Tienda
151,Juliana Torres,juliana.torres151@hotmail.com,3469969206,cali,09/01/2025,64,TIENDA
9,Carlos Suárez,carlos.suarez9@hotmail.com,393 023 7773,Bogota,24/12/2025,50,App
115,  Sofía  Cortés ,sofia.cortes115@outlook.com ,336 462 7128,Medellín,2026-01-08,54,Tienda
80,Sebastián Díaz,SEBASTIAN.DIAZ80@GMAIL.COM,360 949 5707,Medellín,23/04/2025,41,App
201,Andrés Muñoz,andres.munoz41@outlook.com,382 397 6517,Bogotá D.C.,19/10/2025,56,APP
140,David Vargas,david.vargas140@corhuila.edu.co,3647930174,neiva,14/11/2025,65,Tienda
30,NATALIA DÍAZ,natalia.diaz30@outlook.com,3564534276,,19/08/2025,38,App 
4,santiago torres,santiago.torres4@corhuila.edu.co,,Cali ,12/05/2025,67,Tienda
40,Jorge Castro,jorge.castro40@hotmail.com,+57 386-811-0375,medellín,03/03/2025,24 años,web
51,Sebastián López,sebastian.lopez51@outlook.com,+57 325-390-5036,NEIVA ,19/07/2025,53,Tienda 
78,Laura Vargas,laura.vargas78@outlook.com,3732702768,Cali,07/06/2026,35,App
163,Andrés López,andres.lopez163@corhuila.edu.co,3786893859,cali,2026-08-04,66,Tienda
112,Natalia Silva,natalia.silva112@outlook.com,354 436 0348,cali,2026-04-16,27,Tienda 
127,María Muñoz,maria.munoz127@hotmail.com,306 743 1378,NEIVA ,05/09/2025,42,App
55,José Gómez,JOSE.GOMEZ55@GMAIL.COM,+57 394-131-9703,MEDELLIN ,2025-09-17,29,App
123,Felipe Quintero,felipe.quintero123@outlook.com,339 316 7906,cali,19/12/2025,63,Web
91,PAULA DÍAZ,paula.diaz91@outlook.com,3619391778,Bogotá D.C.,29/08/2026,39,Tienda
175,María Pérez,maria.perez175@gmail.com,3268673256,Cali,2025-12-28,69,Tienda
44,ana silva,ana.silva44@corhuila.edu.co,3575525167,cali,2025-04-05,59,Tienda
154,Ana Castro,ANA.CASTRO154@CORHUILA.EDU.CO,+57 353-711-3780,NEIVA ,2026-01-05,57 años,Tienda
49,Carlos Torres,carlos.torres49@hotmail.com,3362295551,Medellin,2026-04-22,20,App
164,MIGUEL QUINTERO,MIGUEL.QUINTERO164@CORHUILA.EDU.CO,(395) 3116963,neiva,2026-06-30,20 años,Tienda
60,  David  Martínez ,david.martinez60@outlook.com,3883781898,CALI,13/06/2025,44,Tienda
54,SEBASTIÁN GARCÍA,sebastian.garcia54@corhuila.edu.co,325 370 0918,MEDELLIN ,15/01/2026,56,Web
21,sebastián rojas,,+57 341-247-2300,medellín,01/09/2026,48,
125,,natalia.cortes125@hotmail.com,379 176 4862,Medellín,2025-06-12,,App
203,david vargas,david.vargas140@corhuila.edu.co,3101,NEIVA ,13/01/2026,65 años,Tienda
199,PAULA SUÁREZ,paula.suarez199@gmail.com,317 603 6388,,2025-11-30,42,App
77,andrés rojas,andres.rojas77@gmail.com,3619906836,cali,2026-06-27,48,App
108,DAVID DÍAZ,david.diaz108@gmail.com ,3743731696,Bogotá D.C.,2025-03-13,31,App
169,ANA ORTIZ,,357 451 5283,Neiva,2025-04-04,18,WEB
13,José Castro,jose.castro13@corhuila.edu.co,,Cali,2026-01-23,25,app
17,Juan Vargas,juan.vargas17@hotmail.com,+57 351-996-5748,Neiva,2025-05-16,42,app
48,ANA DÍAZ,ana.diaz48@gmail.com,3382861015,Cali ,2025-10-26,24,WEB
157,ana ramírez,ana.ramirez157@outlook.com,,Bogota,2025-03-22,32.0,TIENDA
120,Jorge Hernández,,3564806779,Cali,2025-05-29,37,Web
22,Felipe Rodríguez,felipe.rodriguez22@outlook.com,+57 343-213-4569,Neiva,2026-08-13,47,Web
174,Sofía Vargas,sofia.vargas174@corhuila.edu.co,+57 380-246-6816,Cali,14/03/2025,53,App
131,Ana Suárez,ana.suarez131@gmail.com,3515771834,Cali ,2025-12-20,62,web
115,  Sofía  Cortés ,sofia.cortes115@outlook.com ,336 462 7128,Medellín,2026-01-08,54,Tienda
74,CARLOS TORRES,carlos.torres74@gmail.com,31012,neiva,2025-10-11,66 años,APP
31,SOFÍA CORTÉS,sofia.cortes31@gmail.com,3989245130,Neiva,08/11/2025,34,Web
156,SANTIAGO CORTÉS,SANTIAGO.CORTES156@GMAIL.COM,389 814 3988,Cali ,2025-11-04,19,Web
34,Juan Moreno,juan.moreno34@corhuila.edu.co,3275208805,Neiva,30/01/2025,21,Tienda
155,  Valentina  Martínez ,valentina.martinez155@hotmail.com,376 352 5511,Cali,08/10/2025,25,App
178,MARÍA VARGAS,maria.vargas178@hotmail.com,3398702024,BOGOTA,2026-03-22,22,Tienda
190,JUAN DÍAZ,juan.diaz190@gmail.com ,3412722223,Bogotá D.C.,2025-02-13,66,WEB
124,  María  García ,maria.garcia124@corhuila.edu.co,3771993926,medellín,2025-07-10,28,Tienda 
7,Carlos Silva,carlos.silva7@corhuila.edu.co,+57 354-603-3474,CALI,24/10/2025,19,tienda
101,ANA CORTÉS,ANA.CORTES101@GMAIL.COM,+57 364-783-2174,,2026-04-12,70,tienda
150,natalia silva,natalia.silva150@gmail.com ,3061177372,Bogota,08/03/2025,61 años,Web
200,Miguel Martínez,miguel.martinez200@hotmail.com,,bogotá,2025-07-19,,App 
35,  Daniela  Quintero ,DANIELA.QUINTERO35@CORHUILA.EDU.CO,3714551061,Cali ,28/07/2025,67,App 
196,,david.gomez196@gmail.com,3680078166,Neiva,08/08/2025,18,Web
62,sofía muñoz,sofia.munoz62@gmail.com,(310) 5552796,,04/06/2026,22,TIENDA
36,VALENTINA CORTÉS,valentina.cortes36@corhuila.edu.co,314 176 1898,Cali,2026-04-13,32,Tienda 
11,Jorge García,jorge.garcia11@gmail.com,360 400 3086,Cali ,06/11/2025,47,app
206,JULIANA SUÁREZ,juliana.suarez83@corhuila.edu.co,390 918 6546,Medellín,2026-02-05,22,WEB
72,José Ortiz,jose.ortiz72@gmail.com,3907397655,,2025-06-29,46,web
99,natalia garcía,,371 899 1035,medellín,2025-02-08,53,app
94,Felipe Moreno,felipe.moreno94@outlook.com,3528913548,cali,2026-03-14,39,Web
160,María Rodríguez,MARIA.RODRIGUEZ160@HOTMAIL.COM,+57 343-957-1222,Medellín,2025-01-29,21,app
119,Jorge Pérez,JORGE.PEREZ119@GMAIL.COM,(320) 3719932,NEIVA ,16/10/2025,47,web
145,Carlos Ortiz,carlos.ortiz145@hotmail.com,3986608275,CALI,2025-01-16,52,App
2,SEBASTIÁN ORTIZ,sebastian.ortiz2@gmail.com,+57 321-979-2667,Neiva,2025-11-21,52,Tienda
84,,carlos.quintero84@corhuila.edu.co,3062595932,CALI,22/09/2026,52,Tienda
205,  Daniela  Silva ,daniela.silva171@hotmail.com,350 078 0619,neiva,2025-08-02,61 años,web
88,ana cortés,ana.cortes88@outlook.com ,359 797 2380,cali,22/08/2026,62,app
138,Andrés Muñoz,ANDRES.MUNOZ138@OUTLOOK.COM,,NEIVA ,2025-08-02,34.0,
5,Jorge Cortés,jorge.cortes5@outlook.com,3623497554,Bogota,2026-07-22,18,Web
39,SANTIAGO MARTÍNEZ,SANTIAGO.MARTINEZ39@HOTMAIL.COM,(330) 5539400,BOGOTA,2025-04-26,63,Tienda
22,Felipe Rodríguez,felipe.rodriguez22@outlook.com,+57 343-213-4569,Neiva,2026-08-13,47,Web
35,  Daniela  Quintero ,DANIELA.QUINTERO35@CORHUILA.EDU.CO,3714551061,Cali ,28/07/2025,67,App 
113,David Quintero,david.quintero113@outlook.com,3636570659,cali,14/09/2025,45,app
182,CARLOS SUÁREZ,carlos.suarez182@gmail.com,361 406 8481,neiva,2025-02-04,19,TIENDA
184,Santiago Cortés,santiago.cortes184@hotmail.com,3130117932,CALI,2026-01-06,-5,App
153,carlos gómez,carlos.gomez153@outlook.com,329 833 8358,Cali,2026-04-25,,Web
57,JORGE MORENO,jorge.moreno57@gmail.com,,BOGOTA,28/07/2025,44,web
70,María Torres,maria.torres70@corhuila.edu.co,3806982532,cali,2025-11-13,43,App
45,Laura Gómez,laura.gomez45@outlook.com,3440266234,bogotá,2025-06-30,,App 
189,felipe muñoz,felipe.munoz189@hotmail.com,,MEDELLIN ,20/02/2025,68,Tienda
20,MARÍA ROJAS,,326 611 2724,neiva,2025-04-26,53,Tienda
7,Carlos Silva,carlos.silva7@corhuila.edu.co,+57 354-603-3474,CALI,24/10/2025,19,tienda
56,VALENTINA SILVA,valentina.silva56@outlook.com,,Neiva,21/04/2026,0,Web
76,DAVID CASTRO,DAVID.CASTRO76@GMAIL.COM,3755840825,Bogota,2026-07-26,19,WEB
1,Sebastián Martínez,sebastian.martinez1@gmail.com ,363 403 6387,Medellín,2026-07-25,56,Web
173,MARÍA MORENO,maria.moreno173@outlook.com ,3772404487,Bogotá D.C.,2026-07-29,44,web
136,VALENTINA QUINTERO,valentina.quintero136@outlook.com,3455101076,MEDELLIN ,2025-08-01,44,Web
128,ANDRÉS MORENO,andres.moreno128@outlook.com,3006895418,NEIVA ,2025-05-25,150,app
177,CARLOS VARGAS,carlos.vargas177@outlook.com,3312185044,,2026-02-10,56 años,web
69,Juliana Cortés,juliana.cortes69@gmail.com,351 123 9548,Neiva,10/07/2026,19,Web
168,  David  Castro ,david.castro168@outlook.com,353 082 3260,Bogota,2025-08-27,49,App
137,sofía martínez,sofia.martinez137@gmail.com,372 923 7322,Cali,21/04/2025,42,Web
43,Natalia Muñoz,NATALIA.MUNOZ43@CORHUILA.EDU.CO,348 999 3949,Cali,2025-04-03,24,App
111,LAURA VARGAS,laura.vargas111@gmail.com,3678278282,BOGOTA,27/09/2025,61,
90,LAURA PÉREZ,laura.perez90@corhuila.edu.co,394 015 9706,cali,2026-03-21,41,web
73,juan castro,JUAN.CASTRO73@OUTLOOK.COM,3091572650,CALI,2025-04-13,24,App 
24,PAULA GARCÍA,juan perez@mail.com,364 760 2643,Medellín,2025-04-12,70,App
143,Juan Rojas,juan.rojas143@outlook.com,3675135248,Bogotá D.C.,2026-07-09,69,Web
186,carlos vargas,maria@,309 842 6462,BOGOTA,2025-02-25,50.0,Tienda
93,josé garcía,jose.garcia93@corhuila.edu.co,(363) 9621833,Bogotá D.C.,2026-04-01,25,Web
104,Juan Gómez,juan.gomez104@gmail.com,+57 355-356-9863,Bogotá,23/08/2026,35,App 
152,  Paula  Hernández ,paula.hernandez152@corhuila.edu.co,368 959 3590,Bogotá D.C.,08/02/2025,55,Web 
196,,david.gomez196@gmail.com,3680078166,Neiva,08/08/2025,18,Web
68,JUAN CASTRO,juan.castro68@corhuila.edu.co,310 735 4538,Neiva,2026-04-30,40,Web
26,Natalia Cortés,NATALIA.CORTES26@OUTLOOK.COM,,medellín,2025-02-19,,WEB
197,Ana Ortiz,ana.ortiz197@outlook.com,309 709 9244,Medellin,2026-03-30,55,WEB
176,Juliana Ortiz,juliana.ortiz176@gmail.com,3867209450,cali,30/06/2025,,web
```

</details>

### 1.2 Conteo de nulos y otros problemas detectados

**Nulos por columna (antes de limpiar):**

| Columna        |   Nulos antes |
|:---------------|--------------:|
| nombre         |             5 |
| correo         |             9 |
| telefono       |            23 |
| ciudad         |            11 |
| edad           |            17 |
| canal_registro |             6 |
| **Total**      |            71 |

**Resumen de problemas encontrados:**

| # | Problema | Cuántos | Cómo se detectó |
|---|---|---:|---|
| 1 | Nulos | 71 celdas | `df.isna().sum()` |
| 2 | Filas duplicadas exactas | 10 | `df.duplicated().sum()` |
| 3 | Clientes repetidos: mismo correo con otro `cliente_id` (y el correo escrito distinto) | 6 | Normalizar el correo y buscar repetidos |
| 4 | Correos inválidos (sin `@`, sin dominio, etc.) | 6 | Expresión regular |
| 5 | Teléfonos inválidos (muy pocos dígitos) | 5 | Quitar símbolos y contar dígitos |
| 6 | Teléfonos con 7 formatos distintos (`310 123 4567`, `+57 310-123-4567`, `(310) 1234567`...) | 7 formatos | Reemplazar dígitos por `9` y contar patrones |
| 7 | Ciudades escritas de 16 maneras para solo 4 ciudades | 16 variantes | `df["ciudad"].nunique()` |
| 8 | Fechas en 2 formatos, guardadas como texto | 72 en `dd/mm/aaaa` y 144 en `aaaa-mm-dd` | Inspección y `dtypes` |
| 9 | Edad guardada como texto (`"34 años"`, `"34.0"`) | 29 | `dtypes` y búsqueda de texto |
| 10 | Edades fuera de rango (-5, 0, 150) | 3 | Filtro de rango (16 a 90) |
| 11 | Mayúsculas/espacios inconsistentes | 120 nombres, 50 correos, 82 canales | Comparar con la versión normalizada |

### 1.3 Qué decisión se tomó con cada problema

| Problema | Decisión | Resultado |
|---|---|---|
| Filas duplicadas exactas | **Eliminar** (`drop_duplicates()`) | 10 filas menos |
| Correo nulo o inválido | **Eliminar la fila**: sin correo no se puede contactar ni identificar al cliente | 14 filas menos |
| Clientes repetidos (mismo correo) | **Eliminar** y conservar el **primer registro** (el más antiguo) | 6 filas menos |
| `nombre` nulo | **Imputar** con `"Sin nombre"` | 4 filas |
| `telefono` nulo o inválido | **Imputar** con `"No registrado"` (no se puede deducir) | 24 filas |
| `ciudad` nula | **Imputar** con `"No informado"` | 9 filas |
| `canal_registro` nulo | **Imputar** con `"No informado"` | 5 filas |
| `edad` nula o fuera de rango | **Imputar** con la **mediana** (44 años) | 17 filas |
| Texto (nombre, canal, correo, ciudad) | **Normalizar**: quitar espacios, unificar mayúsculas, quitar tildes solo para comparar ciudades | Ver Punto 3 |
| Teléfono | **Normalizar** a 10 dígitos (sin `+57`, espacios ni guiones) | Un solo formato |
| `fecha_registro` | **Corregir el tipo**: texto → `datetime64` | Un solo formato |
| `edad` | **Corregir el tipo**: texto → entero | `int64` |

### 1.4 Script de limpieza en pandas

```python
import re
import unicodedata
import pandas as pd

df = pd.read_csv("clientes_raw.csv")

# ---------- funciones auxiliares ----------
def sin_tildes(t):
    return unicodedata.normalize("NFKD", t).encode("ascii", "ignore").decode()

def clave_correo(s):                       # para detectar clientes repetidos
    return s.astype("string").str.strip().str.lower()

def patron_tel(s):                         # "310 123 4567" -> "999 999 9999"
    return s.dropna().astype(str).map(lambda x: re.sub(r"\d", "9", x)).nunique()

def metricas(d, etapa):
    celdas = d.size
    nulos = int(d.isna().sum().sum())
    m = {
        "Filas": len(d),
        "Nulos (total)": nulos,
        "Celdas completas (%)": round(100 * (celdas - nulos) / celdas, 1),
        "Duplicados exactos": int(d.duplicated().sum()),
        "Clientes repetidos (mismo correo)": int(clave_correo(d.drop_duplicates()["correo"]).dropna().duplicated().sum()),
        "Variantes de ciudad": int(d["ciudad"].dropna().nunique()),
        "Formatos de teléfono": int(patron_tel(d["telefono"])),
        "Formatos de fecha": int(d["fecha_registro"].dropna().astype(str).str.replace(r"\d", "9", regex=True).nunique()),
    }
    print(f"\n===== {etapa} =====")
    for k, v in m.items():
        print(f"{k}: {v}")
    print("Nulos por columna:", d.isna().sum()[d.isna().sum() > 0].to_dict() or "ninguno")
    print("Tipos:", d.dtypes.astype(str).to_dict())
    return m

antes = metricas(df, "ANTES")

# ---------- 1) Duplicados exactos ----------
df = df.drop_duplicates()

# ---------- 2) Normalizar texto ----------
df["nombre"] = (df["nombre"].str.strip().str.replace(r"\s+", " ", regex=True).str.title())
df["nombre"] = df["nombre"].fillna("Sin nombre")                       # imputar
df["canal_registro"] = df["canal_registro"].str.strip().str.title().fillna("No informado")

mapa_ciudad = {"neiva": "Neiva", "bogota": "Bogotá", "bogota d.c.": "Bogotá",
               "medellin": "Medellín", "cali": "Cali"}
df["ciudad"] = (df["ciudad"].str.strip().str.lower().map(sin_tildes, na_action="ignore").map(mapa_ciudad)).fillna("No informado")

# ---------- 3) Correo: normalizar, validar y eliminar los inválidos/nulos ----------
df["correo"] = df["correo"].str.strip().str.lower()
valido = df["correo"].str.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", na=False)
print("\nCorreos inválidos o nulos que se eliminan:", int((~valido).sum()))
df = df[valido]

# ---------- 4) Tipos: fecha, edad, teléfono ----------
iso = pd.to_datetime(df["fecha_registro"], format="%Y-%m-%d", errors="coerce")
dmy = pd.to_datetime(df["fecha_registro"], format="%d/%m/%Y", errors="coerce")
df["fecha_registro"] = iso.fillna(dmy)

edad = pd.to_numeric(df["edad"].astype("string").str.extract(r"(-?\d+)")[0], errors="coerce")
edad = edad.where(edad.between(16, 90))                                # fuera de rango = inválida
df["edad"] = edad.fillna(edad.median()).astype(int)                    # imputar con la mediana

tel = df["telefono"].astype("string").str.replace(r"\D", "", regex=True)
tel = tel.str.replace(r"^57(?=\d{10}$)", "", regex=True)
df["telefono"] = tel.where(tel.str.match(r"^3\d{9}$", na=False)).fillna("No registrado")

# ---------- 5) Clientes repetidos: se conserva el primer registro ----------
df = df.sort_values("fecha_registro").drop_duplicates(subset="correo", keep="first")
df = df.sort_values("cliente_id").reset_index(drop=True)

despues = metricas(df, "DESPUÉS")
df.to_csv("clientes_limpio.csv", index=False)
```

---

## Punto 2: Reporte antes/después

> **Enunciado:** reporta antes/después (nº de filas, nulos, duplicados).

### 2.1 Tabla principal

| Métrica | Antes | Después |
|---|---:|---:|
| **Filas** | **216** | **186** |
| **Nulos (total de celdas)** | **71** | **0** |
| **Duplicados (filas exactas)** | **10** | **0** |
| Clientes repetidos (mismo correo) | 6 | 0 |
| Celdas completas | 95,9 % | 100 % |
| Variantes de ciudad | 16 | 5 (4 ciudades + `No informado`) |
| Formatos de fecha | 2 | 1 |

**Cuentas que explican el cambio de filas:** 216 − 10 duplicados exactos − 14 correos nulos o inválidos − 6 clientes repetidos = **186 filas**.

### 2.2 Nulos por columna

| Columna        |   Nulos antes |   Nulos después |
|:---------------|--------------:|----------------:|
| nombre         |             5 |               0 |
| correo         |             9 |               0 |
| telefono       |            23 |               0 |
| ciudad         |            11 |               0 |
| edad           |            17 |               0 |
| canal_registro |             6 |               0 |
| **Total**      |            71 |               0 |

### 2.3 Tipos de datos

| Columna | Antes | Después |
|---|---|---|
| `fecha_registro` | texto (2 formatos) | `datetime64` |
| `edad` | texto (`"34 años"`, `"34.0"`) | `int64` |
| `telefono` | texto con símbolos y espacios | texto de 10 dígitos |
| `cliente_id` | `int64` | `int64` |
| `nombre`, `correo`, `ciudad`, `canal_registro` | texto sin normalizar | texto normalizado |

### 2.4 Evidencia con filas reales

**Filas con varios problemas (antes):**

|   cliente_id | nombre            | correo                             | telefono         | ciudad   | fecha_registro   | edad    | canal_registro   |
|-------------:|:------------------|:-----------------------------------|:-----------------|:---------|:-----------------|:--------|:-----------------|
|           35 | Daniela  Quintero | DANIELA.QUINTERO35@CORHUILA.EDU.CO | 3714551061       | Cali     | 28/07/2025       | 67      | App              |
|           40 | Jorge Castro      | jorge.castro40@hotmail.com         | +57 386-811-0375 | medellín | 03/03/2025       | 24 años | web              |
|           67 | juan garcía       | JUAN.GARCIA67@OUTLOOK.COM          | 390 748 7169     | NEIVA    | 20/04/2026       | 23      | Web              |
|           88 | ana cortés        | ana.cortes88@outlook.com           | 359 797 2380     | cali     | 22/08/2026       | 62      | app              |
|          103 | NATALIA CORTÉS    | NATALIA.CORTES103@OUTLOOK.COM      | +57 381-289-2217 | MEDELLIN | 27/04/2025       | 50      | Tienda           |

**Las mismas filas después de limpiar:**

|   cliente_id | nombre           | correo                             |   telefono | ciudad   | fecha_registro   |   edad | canal_registro   |
|-------------:|:-----------------|:-----------------------------------|-----------:|:---------|:-----------------|-------:|:-----------------|
|           35 | Daniela Quintero | daniela.quintero35@corhuila.edu.co | 3714551061 | Cali     | 2025-07-28       |     67 | App              |
|           40 | Jorge Castro     | jorge.castro40@hotmail.com         | 3868110375 | Medellín | 2025-03-03       |     24 | Web              |
|           67 | Juan García      | juan.garcia67@outlook.com          | 3907487169 | Neiva    | 2026-04-20       |     23 | Web              |
|           88 | Ana Cortés       | ana.cortes88@outlook.com           | 3597972380 | Cali     | 2026-08-22       |     62 | App              |
|          103 | Natalia Cortés   | natalia.cortes103@outlook.com      | 3812892217 | Medellín | 2025-04-27       |     50 | Tienda           |

**Duplicado exacto (cliente 7 aparecía dos veces, quedó una):**

|   cliente_id | nombre       | correo                        | telefono         | ciudad   | fecha_registro   |   edad | canal_registro   |
|-------------:|:-------------|:------------------------------|:-----------------|:---------|:-----------------|-------:|:-----------------|
|            7 | Carlos Silva | carlos.silva7@corhuila.edu.co | +57 354-603-3474 | CALI     | 24/10/2025       |     19 | tienda           |
|            7 | Carlos Silva | carlos.silva7@corhuila.edu.co | +57 354-603-3474 | CALI     | 24/10/2025       |     19 | tienda           |

**Cliente repetido (mismo correo, dos `cliente_id`). Se conservó el registro más antiguo:**

|   cliente_id | nombre       | correo                     | telefono     | ciudad      | fecha_registro   | edad    | canal_registro   |
|-------------:|:-------------|:---------------------------|:-------------|:------------|:-----------------|:--------|:-----------------|
|           41 | ANDRÉS MUÑOZ | andres.munoz41@outlook.com | NaN          | Bogota      | 2025-07-18       | 56 años | APP              |
|          201 | Andrés Muñoz | andres.munoz41@outlook.com | 382 397 6517 | Bogotá D.C. | 19/10/2025       | 56      | APP              |

Quedó únicamente:

|   cliente_id | nombre       | correo                     | telefono      | ciudad   | fecha_registro   |   edad | canal_registro   |
|-------------:|:-------------|:---------------------------|:--------------|:---------|:-----------------|-------:|:-----------------|
|           41 | Andrés Muñoz | andres.munoz41@outlook.com | No registrado | Bogotá   | 2025-07-18       |     56 | App              |

**Filas eliminadas por correo nulo o inválido (ejemplos):**

|   cliente_id | nombre            | correo          | telefono         | ciudad      | fecha_registro   | edad    | canal_registro   |
|-------------:|:------------------|:----------------|:-----------------|:------------|:-----------------|:--------|:-----------------|
|           66 | valentina silva   | @gmail.com      | NaN              | BOGOTA      | 2026-08-19       | 39 años | Tienda           |
|           86 | ANDRÉS GARCÍA     | NaN             | +57 347-765-5577 | Bogotá D.C. | 2026-06-19       | 19      | App              |
|          110 | Valentina  Moreno | sofia.gmail.com | (300) 5850067    | NaN         | 2025-05-19       | NaN     | App              |
|           99 | natalia garcía    | NaN             | 371 899 1035     | medellín    | 2025-02-08       | 53      | app              |

### 2.5 Salida completa del script

<details>
<summary><b>Ver la salida de la ejecución</b></summary>

```text
===== ANTES =====
Filas: 216
Nulos (total): 71
Celdas completas (%): 95.9
Duplicados exactos: 10
Clientes repetidos (mismo correo): 6
Variantes de ciudad: 16
Formatos de teléfono: 7
Formatos de fecha: 2
Nulos por columna: {'nombre': 5, 'correo': 9, 'telefono': 23, 'ciudad': 11, 'edad': 17, 'canal_registro': 6}
Tipos: {'cliente_id': 'int64', 'nombre': 'str', 'correo': 'str', 'telefono': 'str', 'ciudad': 'str', 'fecha_registro': 'str', 'edad': 'str', 'canal_registro': 'str'}

Correos inválidos o nulos que se eliminan: 14

===== DESPUÉS =====
Filas: 186
Nulos (total): 0
Celdas completas (%): 100.0
Duplicados exactos: 0
Clientes repetidos (mismo correo): 0
Variantes de ciudad: 5
Formatos de teléfono: 2
Formatos de fecha: 1
Nulos por columna: ninguno
Tipos: {'cliente_id': 'int64', 'nombre': 'str', 'correo': 'str', 'telefono': 'string', 'ciudad': 'str', 'fecha_registro': 'datetime64[us]', 'edad': 'int64', 'canal_registro': 'str'}
```

</details>

<details>
<summary><b>Ver el dataset limpio completo (clientes_limpio.csv, 186 filas)</b></summary>

```csv
cliente_id,nombre,correo,telefono,ciudad,fecha_registro,edad,canal_registro
1,Sebastián Martínez,sebastian.martinez1@gmail.com,3634036387,Medellín,2026-07-25,56,Web
2,Sebastián Ortiz,sebastian.ortiz2@gmail.com,3219792667,Neiva,2025-11-21,52,Tienda
3,Miguel Torres,miguel.torres3@hotmail.com,No registrado,Bogotá,2026-02-22,62,Web
4,Santiago Torres,santiago.torres4@corhuila.edu.co,No registrado,Cali,2025-05-12,67,Tienda
5,Jorge Cortés,jorge.cortes5@outlook.com,3623497554,Bogotá,2026-07-22,18,Web
6,Sin nombre,valentina.ortiz6@hotmail.com,3302174254,Bogotá,2026-01-01,51,Web
7,Carlos Silva,carlos.silva7@corhuila.edu.co,3546033474,Cali,2025-10-24,19,Tienda
8,David Rojas,david.rojas8@corhuila.edu.co,3470078573,Medellín,2025-11-11,34,Web
9,Carlos Suárez,carlos.suarez9@hotmail.com,3930237773,Bogotá,2025-12-24,50,App
10,Carlos Cortés,carlos.cortes10@outlook.com,3998962119,Bogotá,2025-01-15,19,Web
11,Jorge García,jorge.garcia11@gmail.com,3604003086,Cali,2025-11-06,47,App
12,Santiago Rojas,santiago.rojas12@outlook.com,No registrado,Medellín,2026-02-07,22,App
13,José Castro,jose.castro13@corhuila.edu.co,No registrado,Cali,2026-01-23,25,App
14,Laura Cortés,laura.cortes14@hotmail.com,3026583693,Neiva,2026-02-18,49,App
15,Felipe Ortiz,felipe.ortiz15@corhuila.edu.co,3142304095,Bogotá,2025-05-26,67,App
16,Jorge Ortiz,jorge.ortiz16@corhuila.edu.co,3873738301,Bogotá,2026-02-22,33,Tienda
17,Juan Vargas,juan.vargas17@hotmail.com,3519965748,Neiva,2025-05-16,42,App
18,Paula Silva,paula.silva18@corhuila.edu.co,3893666654,Neiva,2025-03-06,18,No informado
19,Andrés Quintero,andres.quintero19@gmail.com,3201636172,Neiva,2026-07-31,37,Web
22,Felipe Rodríguez,felipe.rodriguez22@outlook.com,3432134569,Neiva,2026-08-13,47,Web
23,Sebastián Díaz,sebastian.diaz23@hotmail.com,3138924717,Neiva,2026-06-20,25,Tienda
25,María Vargas,maria.vargas25@outlook.com,3556044951,Cali,2025-07-01,44,App
26,Natalia Cortés,natalia.cortes26@outlook.com,No registrado,Medellín,2025-02-19,44,Web
28,Andrés García,andres.garcia28@hotmail.com,3945711464,Neiva,2026-02-06,66,App
29,Sebastián Torres,sebastian.torres29@hotmail.com,3355191538,Medellín,2025-01-10,44,Web
30,Natalia Díaz,natalia.diaz30@outlook.com,3564534276,No informado,2025-08-19,38,App
31,Sofía Cortés,sofia.cortes31@gmail.com,3989245130,Neiva,2025-11-08,34,Web
32,Natalia Rodríguez,natalia.rodriguez32@gmail.com,3259013109,Neiva,2026-01-27,64,Tienda
33,María Ramírez,maria.ramirez33@gmail.com,3949450985,Neiva,2025-10-05,41,Tienda
34,Juan Moreno,juan.moreno34@corhuila.edu.co,3275208805,Neiva,2025-01-30,21,Tienda
35,Daniela Quintero,daniela.quintero35@corhuila.edu.co,3714551061,Cali,2025-07-28,67,App
36,Valentina Cortés,valentina.cortes36@corhuila.edu.co,3141761898,Cali,2026-04-13,32,Tienda
37,Camila Castro,camila.castro37@outlook.com,3734433924,Neiva,2026-04-16,44,App
39,Santiago Martínez,santiago.martinez39@hotmail.com,3305539400,Bogotá,2025-04-26,63,Tienda
40,Jorge Castro,jorge.castro40@hotmail.com,3868110375,Medellín,2025-03-03,24,Web
41,Andrés Muñoz,andres.munoz41@outlook.com,No registrado,Bogotá,2025-07-18,56,App
42,Carlos Díaz,carlos.diaz42@hotmail.com,No registrado,Medellín,2026-03-25,61,App
43,Natalia Muñoz,natalia.munoz43@corhuila.edu.co,3489993949,Cali,2025-04-03,24,App
44,Ana Silva,ana.silva44@corhuila.edu.co,3575525167,Cali,2025-04-05,59,Tienda
45,Laura Gómez,laura.gomez45@outlook.com,3440266234,Bogotá,2025-06-30,44,App
46,Felipe Ortiz,felipe.ortiz46@gmail.com,3137873953,Medellín,2026-07-05,41,Web
47,Camila Quintero,camila.quintero47@hotmail.com,3283309834,Bogotá,2026-08-30,44,Web
48,Ana Díaz,ana.diaz48@gmail.com,3382861015,Cali,2025-10-26,24,Web
49,Carlos Torres,carlos.torres49@hotmail.com,3362295551,Medellín,2026-04-22,20,App
50,Juan Rojas,juan.rojas50@corhuila.edu.co,3306361238,Medellín,2026-01-25,55,App
51,Sebastián López,sebastian.lopez51@outlook.com,3253905036,Neiva,2025-07-19,53,Tienda
52,Sofía Gómez,sofia.gomez52@hotmail.com,3782381936,Bogotá,2025-09-06,37,Web
53,Juliana Cortés,juliana.cortes53@corhuila.edu.co,3590354014,Cali,2026-01-22,33,Tienda
54,Sebastián García,sebastian.garcia54@corhuila.edu.co,3253700918,Medellín,2026-01-15,56,Web
55,José Gómez,jose.gomez55@gmail.com,3941319703,Medellín,2025-09-17,29,App
56,Valentina Silva,valentina.silva56@outlook.com,No registrado,Neiva,2026-04-21,44,Web
57,Jorge Moreno,jorge.moreno57@gmail.com,No registrado,Bogotá,2025-07-28,44,Web
58,Daniela Díaz,daniela.diaz58@corhuila.edu.co,3752807845,No informado,2026-08-09,19,App
59,Valentina López,valentina.lopez59@corhuila.edu.co,3134266013,Cali,2025-09-12,40,Tienda
60,David Martínez,david.martinez60@outlook.com,3883781898,Cali,2025-06-13,44,Tienda
61,José López,jose.lopez61@gmail.com,3311948009,Cali,2025-01-09,30,App
62,Sofía Muñoz,sofia.munoz62@gmail.com,3105552796,No informado,2026-06-04,22,Tienda
63,Sofía Rojas,sofia.rojas63@gmail.com,3331314872,Medellín,2026-07-18,68,Web
64,Camila Gómez,camila.gomez64@corhuila.edu.co,No registrado,Neiva,2025-09-18,34,App
65,Laura Quintero,laura.quintero65@hotmail.com,3265796435,Cali,2025-12-11,47,Web
67,Juan García,juan.garcia67@outlook.com,3907487169,Neiva,2026-04-20,23,Web
68,Juan Castro,juan.castro68@corhuila.edu.co,3107354538,Neiva,2026-04-30,40,Web
69,Juliana Cortés,juliana.cortes69@gmail.com,3511239548,Neiva,2026-07-10,19,Web
70,María Torres,maria.torres70@corhuila.edu.co,3806982532,Cali,2025-11-13,43,App
71,Ana Díaz,ana.diaz71@outlook.com,3352478474,Neiva,2026-09-08,64,Tienda
72,José Ortiz,jose.ortiz72@gmail.com,3907397655,No informado,2025-06-29,46,Web
73,Juan Castro,juan.castro73@outlook.com,3091572650,Cali,2025-04-13,24,App
74,Carlos Torres,carlos.torres74@gmail.com,No registrado,Neiva,2025-10-11,66,App
75,Camila Quintero,camila.quintero75@gmail.com,3291812763,No informado,2026-08-07,42,Web
76,David Castro,david.castro76@gmail.com,3755840825,Bogotá,2026-07-26,19,Web
77,Andrés Rojas,andres.rojas77@gmail.com,3619906836,Cali,2026-06-27,48,App
78,Laura Vargas,laura.vargas78@outlook.com,3732702768,Cali,2026-06-07,35,App
79,Sofía Silva,sofia.silva79@hotmail.com,3712670973,Medellín,2025-06-22,24,App
80,Sebastián Díaz,sebastian.diaz80@gmail.com,3609495707,Medellín,2025-04-23,41,App
81,María Muñoz,maria.munoz81@outlook.com,3432705205,Bogotá,2026-08-06,44,App
82,Jorge Cortés,jorge.cortes82@hotmail.com,3108494584,Medellín,2026-07-27,30,Web
83,Juliana Suárez,juliana.suarez83@corhuila.edu.co,No registrado,Medellín,2025-12-02,22,Web
84,Sin nombre,carlos.quintero84@corhuila.edu.co,3062595932,Cali,2026-09-22,52,Tienda
85,Valentina Cortés,valentina.cortes85@gmail.com,3573204363,Medellín,2025-02-21,70,Web
87,Miguel Moreno,miguel.moreno87@hotmail.com,3900464764,Bogotá,2026-04-18,44,Tienda
88,Ana Cortés,ana.cortes88@outlook.com,3597972380,Cali,2026-08-22,62,App
89,Miguel García,miguel.garcia89@gmail.com,3283759543,Medellín,2025-02-18,69,Tienda
90,Laura Pérez,laura.perez90@corhuila.edu.co,3940159706,Cali,2026-03-21,41,Web
91,Paula Díaz,paula.diaz91@outlook.com,3619391778,Bogotá,2026-08-29,39,Tienda
92,José Moreno,jose.moreno92@hotmail.com,3575063660,Bogotá,2025-04-05,54,App
93,José García,jose.garcia93@corhuila.edu.co,3639621833,Bogotá,2026-04-01,25,Web
94,Felipe Moreno,felipe.moreno94@outlook.com,3528913548,Cali,2026-03-14,39,Web
95,Paula Díaz,paula.diaz95@corhuila.edu.co,3260907748,Neiva,2026-07-09,37,Tienda
96,Laura Díaz,laura.diaz96@gmail.com,3925665965,Neiva,2025-06-01,42,Tienda
97,Daniela Gómez,daniela.gomez97@gmail.com,3519277654,Neiva,2025-06-21,41,App
98,Sebastián Silva,sebastian.silva98@corhuila.edu.co,3547495697,Bogotá,2026-06-20,67,App
100,Camila Vargas,camila.vargas100@outlook.com,3279370357,Bogotá,2025-06-05,63,Web
101,Ana Cortés,ana.cortes101@gmail.com,3647832174,No informado,2026-04-12,70,Tienda
102,Juliana Suárez,juliana.suarez102@corhuila.edu.co,3933581932,Bogotá,2025-01-04,38,Web
103,Natalia Cortés,natalia.cortes103@outlook.com,3812892217,Medellín,2025-04-27,50,Tienda
104,Juan Gómez,juan.gomez104@gmail.com,3553569863,Bogotá,2026-08-23,35,App
105,Jorge Ramírez,jorge.ramirez105@corhuila.edu.co,3230808810,Cali,2026-02-13,48,App
106,David Rojas,david.rojas106@corhuila.edu.co,3400711618,Bogotá,2025-10-31,44,App
107,Sofía Muñoz,sofia.munoz107@gmail.com,No registrado,Cali,2025-08-23,56,App
108,David Díaz,david.diaz108@gmail.com,3743731696,Bogotá,2025-03-13,31,App
109,Juan Silva,juan.silva109@hotmail.com,3167818095,Neiva,2026-05-08,60,App
111,Laura Vargas,laura.vargas111@gmail.com,3678278282,Bogotá,2025-09-27,61,No informado
112,Natalia Silva,natalia.silva112@outlook.com,3544360348,Cali,2026-04-16,27,Tienda
113,David Quintero,david.quintero113@outlook.com,3636570659,Cali,2025-09-14,45,App
114,María Gómez,maria.gomez114@gmail.com,3940297339,Bogotá,2025-03-24,53,Web
115,Sofía Cortés,sofia.cortes115@outlook.com,3364627128,Medellín,2026-01-08,54,Tienda
116,José Ramírez,jose.ramirez116@outlook.com,3449990516,Neiva,2025-02-17,52,App
117,Juliana Rojas,juliana.rojas117@gmail.com,3483672279,Bogotá,2025-05-11,45,Tienda
118,Felipe Vargas,felipe.vargas118@gmail.com,No registrado,Bogotá,2026-03-10,28,Web
119,Jorge Pérez,jorge.perez119@gmail.com,3203719932,Neiva,2025-10-16,47,Web
121,Laura Cortés,laura.cortes121@corhuila.edu.co,3490706937,Cali,2025-11-21,40,Web
122,Paula Suárez,paula.suarez122@hotmail.com,3313679850,Neiva,2025-12-13,51,App
123,Felipe Quintero,felipe.quintero123@outlook.com,3393167906,Cali,2025-12-19,63,Web
124,María García,maria.garcia124@corhuila.edu.co,3771993926,Medellín,2025-07-10,28,Tienda
125,Sin nombre,natalia.cortes125@hotmail.com,3791764862,Medellín,2025-06-12,44,App
126,Daniela Muñoz,daniela.munoz126@hotmail.com,No registrado,Neiva,2026-09-01,54,Web
127,María Muñoz,maria.munoz127@hotmail.com,3067431378,Neiva,2025-09-05,42,App
128,Andrés Moreno,andres.moreno128@outlook.com,3006895418,Neiva,2025-05-25,44,App
129,Felipe López,felipe.lopez129@gmail.com,3236206977,Bogotá,2025-06-19,52,No informado
130,David Gómez,david.gomez130@gmail.com,3698789171,Bogotá,2026-02-19,36,Web
131,Ana Suárez,ana.suarez131@gmail.com,3515771834,Cali,2025-12-20,62,Web
132,Natalia Suárez,natalia.suarez132@gmail.com,No registrado,Bogotá,2025-07-21,44,App
133,Daniela Ortiz,daniela.ortiz133@outlook.com,3681147848,Neiva,2026-03-17,54,Tienda
134,María Ortiz,maria.ortiz134@outlook.com,3883501065,Cali,2025-01-08,62,Tienda
135,Carlos Martínez,carlos.martinez135@corhuila.edu.co,3883819310,Medellín,2025-05-07,18,Web
136,Valentina Quintero,valentina.quintero136@outlook.com,3455101076,Medellín,2025-08-01,44,Web
137,Sofía Martínez,sofia.martinez137@gmail.com,3729237322,Cali,2025-04-21,42,Web
138,Andrés Muñoz,andres.munoz138@outlook.com,No registrado,Neiva,2025-08-02,34,No informado
139,Juliana Cortés,juliana.cortes139@hotmail.com,3663214572,Medellín,2026-07-08,44,Tienda
140,David Vargas,david.vargas140@corhuila.edu.co,3647930174,Neiva,2025-11-14,65,Tienda
141,Juliana Ortiz,juliana.ortiz141@gmail.com,3079528270,Bogotá,2025-11-09,48,Tienda
142,Sofía Ortiz,sofia.ortiz142@corhuila.edu.co,3478098567,Medellín,2026-09-12,52,Web
143,Juan Rojas,juan.rojas143@outlook.com,3675135248,Bogotá,2026-07-09,69,Web
144,Juliana Silva,juliana.silva144@hotmail.com,3614338357,Cali,2025-10-13,59,Tienda
145,Carlos Ortiz,carlos.ortiz145@hotmail.com,3986608275,Cali,2025-01-16,52,App
146,Felipe Hernández,felipe.hernandez146@outlook.com,3110003169,Cali,2025-07-26,64,Tienda
147,Felipe Gómez,felipe.gomez147@gmail.com,3665563873,Medellín,2026-02-24,33,Web
148,Juan Cortés,juan.cortes148@gmail.com,3590528157,Neiva,2025-09-12,25,App
149,María Moreno,maria.moreno149@hotmail.com,3379473631,No informado,2025-08-22,36,App
150,Natalia Silva,natalia.silva150@gmail.com,3061177372,Bogotá,2025-03-08,61,Web
151,Juliana Torres,juliana.torres151@hotmail.com,3469969206,Cali,2025-01-09,64,Tienda
152,Paula Hernández,paula.hernandez152@corhuila.edu.co,3689593590,Bogotá,2025-02-08,55,Web
153,Carlos Gómez,carlos.gomez153@outlook.com,3298338358,Cali,2026-04-25,44,Web
154,Ana Castro,ana.castro154@corhuila.edu.co,3537113780,Neiva,2026-01-05,57,Tienda
155,Valentina Martínez,valentina.martinez155@hotmail.com,3763525511,Cali,2025-10-08,25,App
156,Santiago Cortés,santiago.cortes156@gmail.com,3898143988,Cali,2025-11-04,19,Web
157,Ana Ramírez,ana.ramirez157@outlook.com,No registrado,Bogotá,2025-03-22,32,Tienda
158,Sebastián Castro,sebastian.castro158@outlook.com,3385655956,Bogotá,2025-06-22,25,App
160,María Rodríguez,maria.rodriguez160@hotmail.com,3439571222,Medellín,2025-01-29,21,App
161,Laura Suárez,laura.suarez161@gmail.com,3637936744,Bogotá,2026-03-24,69,Web
162,Juan Silva,juan.silva162@hotmail.com,3203527432,Bogotá,2026-01-16,49,App
163,Andrés López,andres.lopez163@corhuila.edu.co,3786893859,Cali,2026-08-04,66,Tienda
164,Miguel Quintero,miguel.quintero164@corhuila.edu.co,3953116963,Neiva,2026-06-30,20,Tienda
165,Carlos Moreno,carlos.moreno165@corhuila.edu.co,3206884118,Medellín,2026-03-29,60,Tienda
166,Juan Hernández,juan.hernandez166@outlook.com,3981878905,Neiva,2025-04-05,35,Tienda
167,Miguel Rojas,miguel.rojas167@gmail.com,3209480956,Medellín,2025-05-18,43,App
168,David Castro,david.castro168@outlook.com,3530823260,Bogotá,2025-08-27,49,App
170,Ana Torres,ana.torres170@gmail.com,3282970303,Bogotá,2026-05-08,36,Tienda
171,Daniela Silva,daniela.silva171@hotmail.com,No registrado,Neiva,2025-04-25,61,Web
172,Carlos Suárez,carlos.suarez172@gmail.com,3389179256,Bogotá,2025-08-04,37,Tienda
173,María Moreno,maria.moreno173@outlook.com,3772404487,Bogotá,2026-07-29,44,Web
174,Sofía Vargas,sofia.vargas174@corhuila.edu.co,3802466816,Cali,2025-03-14,53,App
175,María Pérez,maria.perez175@gmail.com,3268673256,Cali,2025-12-28,69,Tienda
176,Juliana Ortiz,juliana.ortiz176@gmail.com,3867209450,Cali,2025-06-30,44,Web
177,Carlos Vargas,carlos.vargas177@outlook.com,3312185044,No informado,2026-02-10,56,Web
178,María Vargas,maria.vargas178@hotmail.com,3398702024,Bogotá,2026-03-22,22,Tienda
179,Laura Castro,laura.castro179@outlook.com,3906897808,Bogotá,2025-01-20,18,Tienda
180,Andrés Ortiz,andres.ortiz180@outlook.com,No registrado,Cali,2025-09-20,65,Web
182,Carlos Suárez,carlos.suarez182@gmail.com,3614068481,Neiva,2025-02-04,19,Tienda
183,Daniela Ramírez,daniela.ramirez183@gmail.com,3844205580,Cali,2026-04-03,56,App
184,Santiago Cortés,santiago.cortes184@hotmail.com,3130117932,Cali,2026-01-06,44,App
185,Jorge López,jorge.lopez185@hotmail.com,3165240988,Bogotá,2026-06-01,32,Web
187,Sebastián Rojas,sebastian.rojas187@gmail.com,3447244898,Bogotá,2025-01-15,50,App
188,Juliana Pérez,juliana.perez188@outlook.com,3353070275,Medellín,2025-10-21,38,No informado
189,Felipe Muñoz,felipe.munoz189@hotmail.com,No registrado,Medellín,2025-02-20,68,Tienda
190,Juan Díaz,juan.diaz190@gmail.com,3412722223,Bogotá,2025-02-13,66,Web
191,Valentina Pérez,valentina.perez191@hotmail.com,3028167706,Cali,2025-07-18,61,App
192,Sebastián Díaz,sebastian.diaz192@gmail.com,3944080792,Neiva,2025-12-21,65,Web
193,Miguel López,miguel.lopez193@hotmail.com,No registrado,Cali,2025-05-24,18,Tienda
194,Santiago López,santiago.lopez194@outlook.com,3548893142,Bogotá,2025-06-21,67,App
195,Daniela García,daniela.garcia195@gmail.com,No registrado,Neiva,2026-05-02,52,Web
196,Sin nombre,david.gomez196@gmail.com,3680078166,Neiva,2025-08-08,18,Web
197,Ana Ortiz,ana.ortiz197@outlook.com,3097099244,Medellín,2026-03-30,55,Web
198,Camila Rojas,camila.rojas198@gmail.com,3043729676,Cali,2026-03-07,66,Tienda
199,Paula Suárez,paula.suarez199@gmail.com,3176036388,No informado,2025-11-30,42,App
200,Miguel Martínez,miguel.martinez200@hotmail.com,No registrado,Bogotá,2025-07-19,44,App
```

</details>

---

## Punto 3: Dos dimensiones de calidad que mejoré

> **Enunciado:** comenta 2 dimensiones de calidad que mejoraste.

### Dimensión 1: Unicidad (cada cliente debe aparecer una sola vez)

**Qué pasaba:** había **10 filas duplicadas exactas** y **6 clientes registrados dos veces** con distinto `cliente_id` y el correo escrito diferente (por ejemplo, uno en mayúsculas y otro con un espacio al final). Estos 6 casos no se ven a simple vista: solo aparecen **después de normalizar el correo**, por eso el orden de la limpieza importa.

**Qué hice:** eliminé los duplicados exactos con `drop_duplicates()`; normalicé el correo (`strip()` y `lower()`); y luego quité los repetidos con `drop_duplicates(subset="correo")`, conservando el registro más antiguo.

| | Antes | Después |
|---|---:|---:|
| Filas duplicadas exactas | 10 | 0 |
| Clientes repetidos por correo | 6 | 0 |

**Por qué importa en el caso:** con clientes repetidos, la tienda cuenta más clientes de los que tiene, una misma persona recibe dos veces la misma promoción por correo, y las métricas (clientes por ciudad, por canal) salen infladas.

### Dimensión 2: Consistencia (el mismo dato con el mismo formato)

**Qué pasaba:** el mismo dato venía escrito de muchas formas:

- **Ciudad:** 16 escrituras distintas para solo 4 ciudades (`BOGOTA`, `Bogota`, `Bogotá D.C.`, `bogotá`...).
- **Teléfono:** 7 formatos distintos.
- **Fecha:** 2 formatos, guardados como texto.
- **Texto:** 120 nombres, 82 canales y 50 correos con mayúsculas o espacios inconsistentes.

**Qué hice:** `strip()` y `title()` para nombres y canales; `strip()` y `lower()` para correos; un diccionario de ciudades (sin tildes y en minúscula) que lleva cada variante a su nombre oficial; solo los dígitos y sin `+57` para el teléfono; y `pd.to_datetime()` con los dos formatos para la fecha.

| | Antes | Después |
|---|---:|---:|
| Variantes de ciudad | 16 | 5 (4 ciudades + `No informado`) |
| Formatos de teléfono | 7 | 1 (10 dígitos) + la etiqueta `No registrado` |
| Formatos de fecha | 2 | 1 |

**Evidencia:** al contar clientes por ciudad, antes salían 16 filas para 4 ciudades, con la misma ciudad repartida en varias filas. Después salen las ciudades correctas:

| ciudad       |   clientes |
|:-------------|-----------:|
| Bogotá       |         51 |
| Cali         |         49 |
| Neiva        |         43 |
| Medellín     |         34 |
| No informado |          9 |

**Por qué importa en el caso:** si `BOGOTA` y `Bogotá` cuentan como ciudades distintas, cualquier reporte por ciudad (ventas, clientes, campañas) queda partido y da conclusiones erradas. Con un solo formato, los análisis y los cruces con otras tablas (como las ventas por tienda) funcionan bien.

### Limitaciones

- Se **eliminaron 30 filas** (216 → 186) para asegurar unicidad y correos válidos. Es una decisión consciente: mejor pocos clientes confiables que muchos repetidos o sin forma de contactarlos.
- La **edad imputada con la mediana** (44 años) es una estimación para 17 clientes, no un dato real. Sirve para no perder la fila, pero conviene marcarla si se usa para análisis por edad.
- Al quedarme con el **registro más antiguo** de un cliente repetido puedo perder datos que sí estaban en el otro. Se ve en el cliente 41: su copia (cliente 201) traía el teléfono y el original no, y el teléfono quedó como `No registrado`. Una mejora posible sería combinar ambos registros antes de borrar el repetido.
- También mejoraron la **completitud** (de 95,9 % a 100 % de celdas completas) y la **validez** (correos, teléfonos y edades inválidos), pero los dos casos que se comentan en detalle son unicidad y consistencia.

---

## Cómo ejecutarlo

Copia los dos scripts de los desplegables (`generar_dataset.py` en 1.1 y `limpieza_clientes.py` en 1.4) en archivos con esos nombres y ejecuta:

```bash
pip install pandas
python generar_dataset.py       # crea clientes_raw.csv
python limpieza_clientes.py     # limpia, imprime el antes/después y crea clientes_limpio.csv
```
