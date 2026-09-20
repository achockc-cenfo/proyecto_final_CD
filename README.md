# Proyecto final: Limpieza, análisis exploratorio y K-means geográfico

**Aguas superficiales CONAGUA 2020**  
Informe de decisiones técnicas del equipo

| Campo | Dato |
|---|---|
| Curso / materia | Ciencias de Datos con Python |
| Modalidad | Equipo |
| Profesor | Jorge Ariel Bermúdez Telleria (jbermudez@ucenfotec.ac.cr) |
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
| SEMAFORO | Clasificación Verde / Amarillo / Rojo de la calidad del sitio |
| mg/L | Miligramos por litro (concentración) |

### Qué significan en la práctica

Las siglas de laboratorio no son “variables de Python”. Son mediciones de si el agua está limpia, oxigenada o contaminada. CONAGUA las usa para armar el **semáforo**. K-means **no** las usa: solo agrupa por latitud y longitud.

**Agua superficial.** Es agua que corre o se acumula en la superficie (ríos, lagos, presas, costa). No es agua subterránea (pozos). Este proyecto usa solo el CSV de superficiales 2020.

**Materia orgánica (qué tan “sucia” está el agua)**

- **DBO** (demanda bioquímica de oxígeno): cuánto oxígeno gastan las **bacterias** al descomponer restos orgánicos (aguas residuales, granjas, comida). DBO alta → el río o lago se queda sin oxígeno; peces y plantas sufren. Es el indicador clásico de contaminación biodegradable.
- **DQO** (demanda química de oxígeno): cuánto oxígeno haría falta para oxidar **casi toda** la materia orgánica, también la industrial o poco biodegradable. Casi siempre DQO ≥ DBO. Si DQO es alta y DBO no tanto, apunta más a químicos que a “podredumbre” biológica. En esta base muchos sitios **Rojo** fallan por DQO (aparece en `CONTAMINANTES`).
- **SST** (sólidos suspendidos totales): partículas que enturbian el agua (lodo, arena, materia). Agua turbia, menos luz, peor hábitat. No es lo mismo que DBO/DQO: un río puede ir fangoso por lluvia y aún así no estar “podrido”.

**Bacterias (riesgo sanitario, no “olor a podrido”)**

- **COLI_FEC / CF** (coliformes fecales): bacterias típicas del intestino. Indican contaminación por heces (humanas o de animales). No identifican la especie exacta.
- **E_COLI** (*Escherichia coli*): más específica de origen fecal. Relacionada con enfermedades gastrointestinales si el agua se usa para beber, riego o recreación.
- **ENTEROC** (enterococos fecales): otro indicador fecal. CONAGUA lo usa sobre todo en **agua de mar (COSTERO)**. Por eso en ríos y lagos casi no aparece y en costa sí. Ese hueco no es un error de tipeo.
- **NMP** (número más probable): forma de **contar** bacterias en 100 mL. No es un contaminante; es la unidad (`COLI_FEC_NMP_100mL`).

**Oxígeno y toxicidad**

- **OD / OD_PORC** (oxígeno disuelto, % de saturación): cuánto oxígeno hay disuelto respecto al máximo posible a esa temperatura. Bajo = agua “asfixiada” (anoxia o hipoxia), a menudo ligada a DBO alta. Los peces necesitan ese oxígeno.
- **OD_PORC_SUP / _MED / _FON**: la misma idea en superficie, a media profundidad y en el fondo. En lagos el fondo suele tener menos oxígeno. Medio y fondo casi siempre van vacíos en el CSV; por eso no están en `X_NUMERICAS`.
- **TOX / UT** (toxicidad / unidades de toxicidad): qué tan tóxica es el agua para organismos de prueba (p. ej. *Daphnia* o peces). Casi siempre viene vacía o “no tóxico”; se deja fuera del EDA principal.

**Cómo CONAGUA resume la calidad**

- **SEMAFORO**: Verde (aceptable), Amarillo (indicadores de alerta) o Rojo (contaminada). Es la calidad **ya clasificada** a partir de DBO, DQO, bacterias, oxígeno, etc. En el proyecto es la Y de **validación**, no entra a K-means.
- **CALIDAD_DBO**, **CALIDAD_DQO**, etc.: etiqueta de ese parámetro (Excelente, Buena calidad, Aceptable, Contaminada, Fuertemente contaminada), según las escalas de `Escalas_superficial.csv`.
- **CUMPLE_CON_***: `SI`, `NO` o `ND` por cada criterio. `ND` = no se midió en ese tipo de sitio (no es “cero contaminación”).
- **CONTAMINANTES**: lista de lo que falló (ej. `DQO,CF`). Explica *por qué* un sitio es Rojo.
- **GRUPO**: COSTERO (mar / costa), LÓTICO (río, arroyo), LÉNTICO (lago, presa). No todos miden lo mismo: costa casi no tiene DBO; ríos casi no tienen enterococos.

**Sitio, cuenca y coordenadas**

- **CLAVE / SITIO**: identificador y nombre del punto de monitoreo.
- **CUENCA / CUERPO DE AGUA / SUBTIPO**: a qué río, presa o laguna pertenece el punto.
- **LATITUD y LONGITUD**: dónde está el sitio. Son las **únicas X** de K-means. Latitud ≈ norte-sur; longitud ≈ este-oeste (en México la longitud es negativa).
- **ORGANISMO_DE_CUENCA / ESTADO / MUNICIPIO**: contexto administrativo; no entran al agrupamiento.

**Límites de laboratorio y huecos**

- **LD / LOD** (límite de detección): el aparato no puede medir por debajo de un piso (ej. `<2`). Se sustituye por LD/2 para poder hacer EDA; **no** es la concentración verdadera.
- **`>LD`**: el valor está por encima del tope que reporta el laboratorio. Se deja el número del límite.
- **ND**: no se determinó (no se midió). Se deja como `NaN`; no se inventa un número.
- **NaN**: faltante en pandas. Distinto de “cero contaminación” y distinto de un valor bajo como `<2`.

**Términos del agrupamiento (no miden calidad)**

- **Cluster**: región geográfica que arma K-means. No significa “agua buena” ni “agua mala”.
- **k**: cuántas regiones se piden. Se elige con el codo; no es la Y del problema.
- **Centroide**: punto “centro” de una región (promedio de lat/lon de sus sitios).
- **Inercia**: qué tan apretados quedaron los sitios alrededor de su centro. Baja al subir `k`; se busca el **codo** (donde deja de bajar fuerte). No son kilómetros: lat/lon van escaladas.
- **Silueta**: de −1 a 1. Cerca de 1 = el sitio está nítido en su región; cerca de 0 = frontera; negativo = mejor encajaría en otra. Se reporta; el mapa usa el `k` del codo.

---

## 1. Objetivo y pregunta del enunciado

Este documento registra las decisiones del equipo para el proyecto final. El enunciado pide implementar los conocimientos del curso en un proyecto con datos reales y responder:

> ¿Existe una relación entre la **calidad del agua** y su **ubicación geográfica**, usando K-means sobre **latitud y longitud**?

La rúbrica pide limpieza, EDA (`describe`, boxplot, correlaciones), K-means geográfico, mapa de México y conclusiones. Del Laboratorio 1 se reutiliza el **estilo** (`head`, `sample`, `info`, histogramas, raíz si hay sesgo, Pipeline con `SimpleImputer` y `MinMaxScaler`). El problema **no** es de predicción: no se entrena un modelo supervisado, no hay train/validación/prueba, no se calcula R² ni RMSE y no se usa regresión lineal.

K-means **agrupa** sitios por coordenadas. El semáforo (Verde, Amarillo, Rojo) se usa **después** para validar si esas regiones se parecen en calidad. Eso es ajuste o agrupamiento, no el entrenamiento del Laboratorio 1.

---

## 2. Papel de cada grupo de variables

No todas las columnas del CSV entran al mismo paso. Separarlas evita meter DBO (demanda bioquímica de oxígeno) o el semáforo dentro de K-means, o tratar `X_NUMERICAS` como si fueran las X del enunciado.

| Grupo | Columnas | Papel en el proyecto |
|---|---|---|
| Laboratorio (`COLS_LAB`) | DBO (demanda bioquímica de oxígeno), DQO (demanda química de oxígeno), SST (sólidos suspendidos totales), coliformes, E. coli, enterococos, OD (oxígeno disuelto), toxicidad | Se convierten a número con `a_numero` (`<2`, `ND`, flotantes). Aquí ocurre la mezcla de tipos. |
| EDA (`X_NUMERICAS`; análisis exploratorio) | DBO, DQO, SST, COLI_FEC (coliformes fecales), E_COLI (E. coli), ENTEROC (enterococos), OD_PORC (oxígeno disuelto), OD_PORC_SUP (oxígeno superficial) | Media, mediana, outliers, correlaciones y Pipeline del Lab 1. **No** entran a K-means. |
| Sesgo (`cols_sesgo`) | DBO, DQO, SST, COLI_FEC, E_COLI | Histogramas original vs √ y Pipeline. Se eligen por media >> mediana (análisis, no un umbral automático). |
| Geográficas (`COLS_GEO`) | `LONGITUD`, `LATITUD` | **X del agrupamiento.** Resuelven la parte de ubicación del enunciado. No se imputan. |
| Calidad (`Y`) | `SEMAFORO` | **Validación** de la relación (Verde / Amarillo / Rojo). No se usa para armar los clusters. |
| Contexto | `GRUPO`, `ESTADO`, `CUMPLE_CON_*` | `GRUPO` sirve para comprobar nulos por tipo de agua (COSTERO, LÓTICO, LÉNTICO). No se modelan. |

**Qué resuelve el enunciado:** `LONGITUD` y `LATITUD` (K-means) más `SEMAFORO` (cruce posterior). `X_NUMERICAS` apoyan el EDA; no son las variables del agrupamiento geográfico.

### 2.1 Raíz cuadrada (`cols_sesgo`)

El Laboratorio 1 aplica raíz cuadrada **si hay sesgo**, no a toda columna numérica. `cols_sesgo` es el subconjunto de `X_NUMERICAS` con cola derecha: DBO, DQO, SST, COLI_FEC y E_COLI.

**Cómo se validó.** No hay un `if sesgo > umbral` en el código. Se comparó **media vs mediana** en el `describe` (y, en el script local, una tabla de `sesgo_relativo` = (media − mediana) / mediana). Donde la media es mucho mayor que la mediana hay outliers que jalonean el promedio. Esa decisión se escribió luego como lista fija. Los histogramas original vs √ **ilustran** el efecto; no eligen las columnas.

| Variable | ¿√? | Motivo |
|---|---|---|
| DBO, DQO, SST, coliformes, *E. coli* | Sí | Media >> mediana (sesgo positivo). |
| OD / OD_PORC_SUP | No | Media ≈ mediana; ya es % de saturación. La √ lo distorsionaría. |
| ENTEROC | No | También sesgado, pero casi solo se midió en COSTERO (pocos datos). |
| LONGITUD, LATITUD | No | No son calidad. No se les aplica √. |

La √ se usa en esos histogramas y en el Pipeline de calidad del Lab 1. **No** entra a K-means.

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
