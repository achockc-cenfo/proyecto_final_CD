# Proyecto final: Limpieza, análisis exploratorio y K-means geográfico

**Aguas superficiales CONAGUA 2020**  
Informe de decisiones técnicas del equipo

| Campo | Dato |
|---|---|
| Curso / materia | Ciencias de Datos con Python |
| Modalidad | Equipo |
| Profesor | Jorge Ariel Bermúdez Telleria | jbermudez@ucenfotec.ac.cr |
| Dataset | Datos de calidad del agua de sitios de monitoreo de aguas superficiales 2020 (CONAGUA) |
| Fecha | 19 de septiembre de 2026 |

### Integrantes

| Nombre | Correo |
|---|---|
| Charles Quesada Sandi | cquesadasa@ucenfotec.ac.cr |
| Aaron Chock Chock | achockc@ucenfotec.ac.cr |
| Marvin Jesús Calvo Acuña | mcalvoa@ucenfotec.ac.cr |

### Abreviaturas

| Sigla | Significado |
|---|---|
| CONAGUA | Comisión Nacional del Agua |
| DBO | Demanda Bioquímica de Oxígeno |
| DQO | Demanda Química de Oxígeno |
| SST | Sólidos Suspendidos Totales |
| COLI_FEC / CF | Coliformes fecales |
| E_COLI | Escherichia coli |
| ENTEROC | Enterococos fecales |
| OD / OD_PORC | Oxígeno disuelto (porcentaje de saturación) |
| OD_PORC_SUP | Oxígeno disuelto en superficie |
| TOX / UT | Toxicidad / Unidades de Toxicidad |
| NMP | Número Más Probable (conteo bacteriano) |
| LD / LOD | Límite de detección (Limit of Detection) |
| ND | No determinado / no detectado (no se midió) |
| NaN | Not a Number (valor faltante en pandas) |
| CSV | Archivo de valores separados por comas |
| EDA | Análisis Exploratorio de Datos |
| IQR | Rango intercuartílico (regla de outliers 1.5*IQR) |
| R2 | Coeficiente de determinación (no se usa en este proyecto) |
| RMSE | Raíz del error cuadrático medio (no se usa en este proyecto) |
| K-means | Agrupamiento por k medias (no supervisado) |
| COSTERO | Cuerpo de agua de costa / mar |
| LÓTICO | Agua corriente (ríos, arroyos) |
| LÉNTICO | Agua quieta (lagos, presas) |

---

## 1. Objetivo y pregunta del enunciado

Este documento registra las decisiones del equipo para el proyecto final. El enunciado pide aplicar lo visto en el laboratorio (inspección, limpieza, `describe`, boxplot, correlaciones y Pipeline) a datos reales y responder:

> ¿Existe una relación entre la **calidad del agua** y su **ubicación geográfica**, usando K-means sobre **latitud y longitud**?

Se sigue el **estilo** del Laboratorio 1 (`head`, `sample`, `info`, histogramas, `describe`, raíz cuadrada si hay sesgo, Pipeline con `SimpleImputer` y `MinMaxScaler`). El problema **no** es de predicción: no se entrena un modelo supervisado, no hay train/validación/prueba, no se calcula R² ni RMSE y no se usa regresión lineal.

K-means **agrupa** sitios por coordenadas. El semáforo (Verde, Amarillo, Rojo) se usa **después** para validar si esas regiones se parecen en calidad. Eso es ajuste o agrupamiento, no el entrenamiento del Laboratorio 1.

---

## 2. Papel de cada grupo de variables

No todas las columnas del CSV entran al mismo paso. Separarlas evita meter DBO (demanda bioquímica de oxígeno) o el semáforo dentro de K-means, o tratar `X_NUMERICAS` como si fueran las X del enunciado.

| Grupo | Columnas | Papel en el proyecto |
|---|---|---|
| Laboratorio (`COLS_LAB`) | DBO (demanda bioquímica de oxígeno), DQO (demanda química de oxígeno), SST (sólidos suspendidos totales), coliformes, E. coli, enterococos, OD (oxígeno disuelto), toxicidad | Se convierten a número con `a_numero` (`<2`, `ND`, flotantes). Aquí ocurre la mezcla de tipos. |
| EDA (`X_NUMERICAS`; análisis exploratorio) | DBO, DQO, SST, COLI_FEC (coliformes fecales), E_COLI (E. coli), ENTEROC (enterococos), OD_PORC (oxígeno disuelto), OD_PORC_SUP (oxígeno superficial) | Media, mediana, outliers, correlaciones y Pipeline del Lab 1. **No** entran a K-means. |
| Geográficas (`COLS_GEO`) | `LONGITUD`, `LATITUD` | **X del agrupamiento.** Resuelven la parte de ubicación del enunciado. No se imputan. |
| Calidad (`Y`) | `SEMAFORO` | **Validación** de la relación (Verde / Amarillo / Rojo). No se usa para armar los clusters. |
| Contexto | `GRUPO`, `ESTADO`, `CUMPLE_CON_*` | `GRUPO` sirve para comprobar nulos por tipo de agua (COSTERO, LÓTICO, LÉNTICO). No se modelan. |

**Qué resuelve el enunciado:** `LONGITUD` y `LATITUD` (K-means) más `SEMAFORO` (cruce posterior). `X_NUMERICAS` apoyan el EDA; no son las variables del agrupamiento geográfico.

---

## 3. Limpieza de datos

### 3.1 El archivo CSV no se modifica

`pd.read_csv` carga una copia en memoria (`df_crudo`). La limpieza crea otro DataFrame (`df`). El archivo en disco o en Drive no se sobrescribe.

- `df_crudo`: estado original (`head`, nulos antes, celdas `<2`).
- `df`: versión lista para EDA y K-means.

### 3.2 Búsqueda de nulos

Se cuenta `isna().sum()` por columna. Se muestran solo las que tienen al menos un nulo y su porcentaje. El mismo conteo se hace **antes** y **después** de convertir.

### 3.3 Filas que no son un sitio

Se eliminan registros sin `CLAVE` o con `CLAVE` vacía (filas en blanco al final del CSV).

### 3.4 Datos mal escritos y sustitución (datos censurados)

Las columnas de laboratorio mezclan texto y número: `6`, `4.26`, `<2`, `ND`. La técnica es **sustitución de datos censurados** (LOD = Limit of Detection, límite de detección).

| Caso en el CSV | Decisión | Técnica | Justificación |
|---|---|---|---|
| `ND` o celda vacía | `NaN` | No imputar | No se midió. No se inventa un número. |
| `<2`, `<10`, `<3` | LD / 2 | Sustitución LOD/2 (censura izquierda) | Convención de EDA. No es la medición real. |
| `>100` | El número del límite | Sustitución por el límite (censura derecha) | Permite EDA. Se pierde que era mayor que. |
| `6`, `4.26` | Se deja como `float` | Conversión numérica | Ya es una medición. |

Ejemplo: en `DBO_mg/L`, `<2` pasa a `1.0`. Un `4.26` se queda. LOD/2 (límite de detección entre 2) **no** se aplica a toda la base: solo a celdas de `COLS_LAB` que empiezan con `<`. Semáforo, estado y `CUMPLE_CON_*` no se convierten así. Lat/lon solo se pasan a `float`.

### 3.5 Otra fórmula: LOD / sqrt(2) (límite de detección entre raíz de 2)

También existe **LOD/sqrt(2)** (aproximadamente 0.707 x LD), usada a veces si se supone lognormalidad. En este proyecto se aplica **LOD/2** por simplicidad: si el laboratorio reporta <2, el valor usado es 1. Ninguna fórmula es la concentración verdadera. No se implementó LOD/sqrt(2).

La sustitución es válida como convención de limpieza para el curso. No es Kaplan-Meier ni ROS; el enunciado no los pide.

---

## 4. Eliminación de nulos (selectiva)

No se borra toda fila con algún NaN ni toda columna con un faltante.

| Nulo | Decisión | Motivo |
|---|---|---|
| `LATITUD` o `LONGITUD` faltante | Eliminar la fila | K-means no puede agrupar sin coordenadas. **No se imputan.** |
| `SEMAFORO` faltante | Eliminar la fila | Sin calidad no se valida la relación. **No se imputa.** |
| DBO (demanda bioquímica), DQO (demanda química), enterococos, oxígeno, etc. | Dejar `NaN` | Ese `GRUPO` puede no medir ese parámetro. Se comprueba después. |

### 4.1 Nulos por `GRUPO`

Después de convertir se calcula el % de nulos de `X_NUMERICAS` por COSTERO (costa) / LÓTICO (ríos) / LÉNTICO (lagos o presas). En estos datos: en COSTERO falta DBO en ~91 % de sitios; en LÓTICO y LÉNTICO faltan enterococos en ~100 % y 99 %. En COSTERO los enterococos sí se midieron. No es un error de tipeo: no todos los grupos se evalúan con los mismos parámetros.

---

## 5. Relación con el Laboratorio 1

El laboratorio lista cuatro formas de tratar nulos. No se usan las cuatro.

| Opción del Lab 1 | ¿Se usó? | Decisión del equipo |
|---|---|---|
| Eliminar toda fila con al menos un nulo | No | Se perderían costeros (sin DBO, demanda bioquímica) y muchos ríos (sin enterococos). |
| Eliminar toda columna con al menos un nulo | No | Se irían DBO, DQO, coliformes y oxígeno. |
| Imputar con media / mediana / constante | Solo en el Pipeline de calidad | Mediana + `MinMaxScaler`. Esa matriz **no** entra a K-means. |
| Imputar y agregar columna flag | No | El enunciado no lo pide. |

Lat/lon se escalan aparte con MinMax **sin imputar**, para que un eje no domine. La inercia **no** son kilómetros.

Se copia el patrón del Lab 1. Se cambia la herramienta: el lab predice un número; este proyecto agrupa por ubicación y compara calidad.

---

## 6. Cómo se resuelve el enunciado (dos pasos)

La calidad **no** entra a K-means. Si el semáforo o el DBO (demanda bioquímica de oxígeno) entraran al `fit`, el cruce posterior sería circular.

### 6.1 Paso 1. Agrupar solo por ubicación

K-means recibe únicamente `LONGITUD` y `LATITUD` escaladas. Se prueban k = 2 a 10. El *k* del mapa se elige por el **codo**; se reporta silueta. *k* = 3 es solo comparación: tres colores no implican tres regiones.

Resultado: etiqueta `cluster` y centroides. Cada cluster es una **región**, no agua buena/mala.

### 6.2 Paso 2. Validar con el semáforo

`crosstab(cluster, SEMAFORO)` en conteos y en % dentro de cada cluster.

| Lo que se observa | Interpretación |
|---|---|
| Un cluster muy rojo y otro muy verde | Hay patrón geográfico de calidad. |
| Porcentajes parecidos en todos los clusters | La calidad no se explica solo con lat/lon. |

Son **diferencias observadas**, no una prueba estadística ni un umbral fijo.

Dos mapas de los mismos puntos: uno por cluster (ubicación) y otro por semáforo (calidad). Si se parecen, van juntas; si el semáforo está salpicado, lat/lon no bastan.

---

## 7. Cierre

Se ajustó K-means para agrupar por coordenadas. **No** se entrenó un predictor. La relación calidadubicación se obtiene **después**, al cruzar cada región con el semáforo.

> K-means no clasifica calidad; clasifica sitios por coordenadas. Después comparamos el semáforo dentro de cada región. Si los porcentajes de verde y rojo cambian entre clusters, la calidad está ligada a la ubicación. Si no cambian, latitud y longitud no bastan para explicar la calidad del agua.

El CSV original permanece intacto. Las mediciones de laboratorio quedaron numéricas con LOD/2 (mitad del límite de detección) (y el límite en valores `>LD`). Los nulos de calidad se interpretaron con `GRUPO`. El Pipeline del Laboratorio 1 quedó como preparación de calidad y no se usó para agrupar.

---

## Archivos del proyecto

| Ruta | Contenido |
|---|---|
| `DataSets/` | CSV de sitios y escalas CONAGUA 2020 |
| `solucion_proyecto_final/proyectofinal.py` | Script local (gráficas en `graficas/`) |
| `solucion_proyecto_final/Proyecto_Final_Aguas_Superficiales_Colab.ipynb` | Notebook para Google Colab |
| `decisiones _grupo/Informe_decisiones_proyecto_final.docx` | Este mismo informe en Word |
| `decisiones _grupo/decisiones _limpieza_datos.docx` | Bitácora corta de limpieza |
