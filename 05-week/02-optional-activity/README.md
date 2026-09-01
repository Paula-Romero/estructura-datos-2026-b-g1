# Proyecto de Ciencia de Datos: Análisis de la Línea de Empaque L2

**Programa:** Ingeniería Industrial  
**Asignatura:** Ciencia de Datos  
**Periodo:** 2026-B  
**Corte:** 1  
**Estudiante:** PAULA XIMENA ROMERO VILLEGAS

---

## Descripción general del proyecto

Este proyecto se desarrolla a partir del caso de una **planta agroindustrial del Huila dedicada al tostado y empaque de café**, específicamente en la **línea de empaque L2**.

La línea genera información proveniente de sensores, sistemas de mantenimiento, producción, calidad y condiciones ambientales. El propósito del proyecto es integrar estos datos para comprender el comportamiento de la línea, identificar las causas de las paradas no programadas, estimar el riesgo de futuras fallas y apoyar la toma de decisiones relacionadas con el mantenimiento y la producción.

Este README reúne y organiza los principales fundamentos desarrollados durante las semanas 1 a 4 del primer corte.

---

# 1. Pregunta de negocio y decisión esperada

## Pregunta de negocio

**¿Cómo pueden analizarse los datos generados por la línea de empaque L2 para identificar las causas de las paradas no programadas y anticipar posibles fallas que afecten la continuidad de la producción?**

La línea de empaque puede presentar paradas no programadas que generan pérdida de tiempo productivo, retrasos y costos adicionales. Para comprender este problema es necesario analizar conjuntamente la información de sensores, mantenimiento, producción, calidad y condiciones ambientales.

## Decisión esperada

La información obtenida debe permitir:

- Priorizar equipos o componentes con mayor riesgo de falla.
- Programar actividades de mantenimiento de manera más oportuna.
- Reducir las paradas no programadas.
- Disminuir el tiempo de inactividad.
- Apoyar la planificación de producción y mantenimiento con información basada en datos.

---

# 2. Fuentes y clasificación de datos + V relevantes

## 2.1. Fuentes de datos

| Fuente de datos | Información generada | Clasificación |
|---|---|---|
| Sensores IoT | Temperatura, vibración, presión y otras mediciones | Estructurados / series de tiempo |
| PLC y SCADA | Estados de máquina, eventos y señales operativas | Semiestructurados |
| CMMS | Órdenes de trabajo, fallas y mantenimiento | Estructurados |
| ERP de producción | Cantidad producida, referencias y tiempos | Estructurados |
| Calidad | Inspecciones y controles del producto | Estructurados |
| Condiciones ambientales | Temperatura y humedad | Estructurados / series de tiempo |
| Logs técnicos | Eventos y registros de sistemas | Semiestructurados o no estructurados |

## 2.2. Clasificación general

### Datos estructurados

Son datos organizados en tablas con campos definidos, como órdenes de mantenimiento, duración de paradas, cantidad producida y resultados de inspecciones.

### Datos semiestructurados

Poseen cierta organización, pero no necesariamente se almacenan en tablas tradicionales. En este caso incluyen eventos de PLC, SCADA y archivos de logs.

### Datos no estructurados

No siguen una estructura fija. En el proyecto podrían incorporarse observaciones de técnicos, informes escritos o fotografías de componentes.

## 2.3. V relevantes del Big Data

### Volumen

Los sensores, eventos y registros históricos generan una cantidad creciente de información que requiere una estrategia adecuada de almacenamiento.

### Velocidad

Los sensores y sistemas operativos pueden generar datos continuamente, por lo que es importante recibir y procesar la información con rapidez.

### Variedad

Existen datos numéricos de sensores, tablas de producción y mantenimiento, registros de eventos y posibles observaciones en texto.

### Veracidad

Pueden existir valores erróneos, registros duplicados, datos faltantes o inconsistencias. Por eso los datos deben validarse y limpiarse.

### Valor

Los datos generan valor cuando permiten comprender las paradas, identificar patrones de falla y apoyar decisiones de mantenimiento.

---

# 3. Arquitectura de datos propuesta

**Flujo general:**

**Fuentes de datos → Ingesta → Almacenamiento → Procesamiento → Análisis y BI**

## 3.1. Fuentes de datos

La arquitectura inicia con sensores IoT, PLC, SCADA, CMMS, ERP, sistemas de producción, calidad, condiciones ambientales y logs técnicos.

## 3.2. Ingesta

Se propone un enfoque híbrido:

- **Streaming:** para sensores y eventos continuos.
- **Batch:** para datos históricos de mantenimiento, producción, calidad y otros sistemas.

Una herramienta candidata es **Apache Kafka** para el manejo de eventos continuos.

## 3.3. Almacenamiento

Se propone un **Data Lake o Lakehouse** debido a la variedad de formatos y fuentes.

- **Bronze:** datos crudos.
- **Silver:** datos limpios, validados e integrados.
- **Gold:** datos preparados para análisis e indicadores.

## 3.4. Procesamiento

Durante esta etapa se realizan:

- Limpieza y validación.
- Eliminación de duplicados.
- Manejo de valores faltantes.
- Integración de fuentes.
- Transformación de variables.
- Preparación para análisis.

Una herramienta candidata es **Apache Spark**, útil para datos históricos y continuos.

## 3.5. Análisis y BI

Los datos procesados pueden visualizarse con **Microsoft Power BI** mediante indicadores de:

- Número y duración de paradas.
- Comportamiento de sensores.
- Tendencias de fallas.
- Nivel de riesgo.
- Equipos con mayor número de incidentes.

---

# 4. Tipos de analítica

## 4.1. Analítica descriptiva

**Pregunta:** ¿Cuántas paradas no programadas tuvo la línea L2 durante el último mes y cuánto tiempo total estuvo detenida?

**Justificación:** Es descriptiva porque resume lo que ya ocurrió mediante registros históricos de paradas, fechas, duración y turnos.

**Responde:** **¿Qué pasó?**

---

## 4.2. Analítica diagnóstica

**Pregunta:** ¿Por qué aumentaron las paradas no programadas de la línea L2 durante determinados turnos o periodos?

**Justificación:** Busca identificar posibles causas relacionando las paradas con variables como vibración, temperatura, mantenimiento realizado, producción y condiciones ambientales.

**Responde:** **¿Por qué pasó?**

---

## 4.3. Analítica predictiva

**Pregunta:** ¿Cuál es la probabilidad de que la línea L2 presente una parada no programada en las próximas 72 horas?

**Justificación:** Utiliza información histórica y actual para estimar un evento futuro y calcular un nivel de riesgo.

**Responde:** **¿Qué podría pasar?**

---

## 4.4. Analítica prescriptiva

**Pregunta:** Si existe una alta probabilidad de falla en un componente, ¿qué acción de mantenimiento debería realizarse y en qué momento para reducir el riesgo sin afectar innecesariamente la producción?

**Justificación:** Busca recomendar una decisión considerando el riesgo, personal disponible, repuestos y el impacto de detener la producción.

**Responde:** **¿Qué se debería hacer?**

---

# 5. Uso de Machine Learning

## Machine Learning supervisado

Para la analítica predictiva se propone principalmente **Machine Learning supervisado** porque pueden utilizarse datos históricos en los que ya se conoce el resultado:

- Periodos en los que ocurrió una parada.
- Periodos en los que no ocurrió una parada.

Estas etiquetas permiten entrenar un modelo utilizando variables como temperatura, vibración, presión, tiempo desde el último mantenimiento e historial de fallas.

## Machine Learning no supervisado

Puede utilizarse como complemento para detectar comportamientos anómalos o patrones inusuales cuando no existen etiquetas claras. Sin embargo, para predecir una parada futura, el aprendizaje supervisado es el enfoque más directamente relacionado con el objetivo.

---

# 6. Riesgo ético y posible sesgo

## Riesgo identificado

Un posible riesgo es el **sesgo en los datos históricos de mantenimiento y producción**. Algunos turnos pueden tener registros más completos que otros o ciertas fallas pueden haberse documentado de manera diferente.

Si un modelo aprende de datos incompletos o desequilibrados, podría generar conclusiones equivocadas sobre el riesgo de falla.

## Estrategias de mitigación

- **Validar la calidad y cobertura de los datos:** revisar periodos, turnos y equipos con información incompleta.
- **Estandarizar el registro de fallas:** utilizar categorías y criterios claros.
- **Evitar variables innecesarias:** no utilizar información personal que no sea necesaria para el análisis técnico.
- **Evaluar el modelo en diferentes condiciones:** comprobar su desempeño entre equipos y periodos.
- **Mantener supervisión humana:** las recomendaciones deben apoyar al personal técnico.
- **Documentar las decisiones:** registrar datos y criterios utilizados para facilitar transparencia y revisión.

---

# 7. Conclusión

El proyecto integra fundamentos de Ciencia de Datos y Big Data aplicados a la línea de empaque L2. La información de sensores, mantenimiento, producción, calidad y ambiente puede integrarse en una arquitectura que permita almacenar, procesar y analizar los datos.

Los cuatro tipos de analítica se complementan: la descriptiva muestra lo ocurrido, la diagnóstica ayuda a comprender las causas, la predictiva estima riesgos futuros y la prescriptiva orienta las acciones.

Para la predicción de paradas, el Machine Learning supervisado es el enfoque principal debido a la disponibilidad de registros históricos relacionados con eventos conocidos. Todo el proceso debe considerar la calidad de los datos, los posibles sesgos y la supervisión humana para garantizar decisiones responsables.

---

# Referencias

1. Apache Kafka. (2026). *Apache Kafka Documentation*. https://kafka.apache.org/documentation/
2. Apache Spark. (2026). *Spark Structured Streaming*. https://spark.apache.org/streaming/
3. IBM. (2024). *What is machine learning?* https://www.ibm.com/think/topics/machine-learning
4. Microsoft. (2026). *¿Qué es Power BI?* https://learn.microsoft.com/es-es/power-bi/fundamentals/power-bi-overview
5. NIST. (2023). *Artificial Intelligence Risk Management Framework (AI RMF 1.0)*. https://www.nist.gov/itl/ai-risk-management-framework
6. UNESCO. (2021). *Recommendation on the Ethics of Artificial Intelligence*. https://unesdoc.unesco.org/ark:/48223/pf0000381137
