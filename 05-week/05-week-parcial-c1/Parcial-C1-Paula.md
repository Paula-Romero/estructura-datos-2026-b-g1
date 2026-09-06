## Ciencia de Datos
## Paula Ximena Romero Villegas 

# Actividad: Análisis de datos en una tienda en línea

## Caso seleccionado

Una **tienda en línea** que registra información de sus ventas, clientes y productos. 
Estos datos pueden utilizarse para conocer el comportamiento de las ventas y también
para anticipar qué productos podrían venderse más en el futuro.

------------------------------------------------------------------------

## 1. Tipos de datos

En la tienda en línea se pueden encontrar los siguientes cuatro tipos de
datos:

  -----------------------------------------------------------------------
  Dato                    Ejemplo                 Clasificación
  ----------------------- ----------------------- -----------------------
  Registro de ventas      Fecha, producto,        **Estructurado**
                          cantidad y precio de    
                          cada compra             

  Información de clientes Nombre, correo          **Estructurado**
                          electrónico y dirección 

  Comentarios de clientes Opiniones escritas      **No estructurado**
                          sobre los productos     

  Registros de navegación Datos en formato JSON   **Semiestructurado**
                          sobre las páginas       
                          visitadas, clics y      
                          productos consultados   
  -----------------------------------------------------------------------

Los datos **estructurados** están organizados en tablas y tienen un
formato definido. Los datos **semiestructurados** tienen cierta
organización mediante etiquetas o campos, como ocurre con los archivos
JSON. Por otro lado, los datos **no estructurados** no siguen un formato
fijo, como los comentarios escritos por los clientes.

------------------------------------------------------------------------

## 2. Preguntas de analítica

### Analítica descriptiva

**Pregunta:**\
¿Cuáles fueron los productos más vendidos y cuánto dinero generaron las
ventas durante el último mes?

La analítica descriptiva permite revisar los datos históricos y conocer
qué ocurrió en el negocio.

### Analítica predictiva

**Pregunta:**\
¿Qué productos tienen mayor probabilidad de venderse más durante el
próximo mes?

La analítica predictiva utiliza los datos históricos para identificar
patrones y realizar una estimación sobre lo que podría ocurrir en el
futuro.

------------------------------------------------------------------------

## 3. Diagrama del proceso

``` text
┌──────────────┐
│   FUENTE     │
│ Ventas,      │
│ clientes y   │
│ navegación   │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ALMACENAMIENTO │
│ Base de datos │
│ y archivos    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   ANÁLISIS   │
│ Descriptivo  │
│ y predictivo │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│VISUALIZACIÓN │
│ Gráficos,    │
│ tablas e     │
│ indicadores  │
└──────────────┘
```

### Explicación

Primero se **obtienen los datos** desde las ventas, los clientes y la
navegación de la tienda. Después, los datos se **almacenan** en una base
de datos o en archivos. Luego se realiza el **análisis** para encontrar
información útil y patrones. Finalmente, los resultados se presentan
mediante **gráficos, tablas e indicadores**, lo que facilita la toma de
decisiones.

------------------------------------------------------------------------

## 4. Difference between descriptive analytics and predictive analytics

**Descriptive analytics explains what happened in the past, while
predictive analytics uses historical data to estimate what may happen in
the future.**

**For example, descriptive analytics can show the best-selling products,
while predictive analytics can forecast which products are likely to
sell more next month.**

------------------------------------------------------------------------

## Conclusión

El análisis de datos permite que una tienda en línea convierta la
información de sus operaciones en conocimiento útil. La analítica
descriptiva ayuda a entender lo que ya ocurrió, mientras que la
analítica predictiva permite anticipar posibles situaciones y apoyar la
toma de decisiones.


## Referencias

- IBM. (2024). *¿Qué es el análisis predictivo?* https://www.ibm.com/es-es/think/topics/predictive-analytics
- IBM. (2024). *¿Qué es el analytics de IA?* https://www.ibm.com/es-es/think/topics/ai-analytics
- SAS. (2026). *Analítica predictiva: Qué es y por qué es importante.* https://www.sas.com/es_co/insights/analytics/predictive-analytics.html