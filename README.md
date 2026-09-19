# Proyecto final: Limpieza, an�lisis exploratorio y K-means geogr�fico

**Aguas superficiales CONAGUA 2020**  
Informe de decisiones t�cnicas del equipo

| Campo | Dato |
|---|---|
| Curso / materia | Ciencias de Datos con Python |
| Modalidad | Equipo |
| Profesor | Jorge Ariel Berm�dez Telleria |
| Dataset | Datos de calidad del agua de sitios de monitoreo de aguas superficiales 2020 (CONAGUA) |
| Fecha | 19 de septiembre de 2026 |

### Integrantes

| Nombre | Correo |
|---|---|
| Charles Quesada Sandi | cquesadasa@ucenfotec.ac.cr |
| Aaron Chock Chock | achockc@ucenfotec.ac.cr |
| Marvin Jes�s Calvo Acu�a | mcalvoa@ucenfotec.ac.cr |

---

## 1. Objetivo y pregunta del enunciado

Este documento registra las decisiones del equipo para el proyecto final. El enunciado pide aplicar lo visto en el laboratorio (inspecci�n, limpieza, `describe`, boxplot, correlaciones y Pipeline) a datos reales y responder:

> �Existe una relaci�n entre la **calidad del agua** y su **ubicaci�n geogr�fica**, usando K-means sobre **latitud y longitud**?

Se sigue el **estilo** del Laboratorio 1 (`head`, `sample`, `info`, histogramas, `describe`, ra�z cuadrada si hay sesgo, Pipeline con `SimpleImputer` y `MinMaxScaler`). El problema **no** es de predicci�n: no se entrena un modelo supervisado, no hay train/validaci�n/prueba, no se calcula R� ni RMSE y no se usa regresi�n lineal.

K-means **agrupa** sitios por coordenadas. El sem�foro (Verde, Amarillo, Rojo) se usa **despu�s** para validar si esas regiones se parecen en calidad. Eso es ajuste o agrupamiento, no el entrenamiento del Laboratorio 1.

---

## 2. Papel de cada grupo de variables

No todas las columnas del CSV entran al mismo paso. Separarlas evita meter DBO o el sem�foro dentro de K-means, o tratar `X_NUMERICAS` como si fueran las X del enunciado.

| Grupo | Columnas | Papel en el proyecto |
|---|---|---|
| Laboratorio (`COLS_LAB`) | DBO, DQO, SST, coliformes, E. coli, enterococos, OD, toxicidad | Se convierten a n�mero con `a_numero` (`<2`, `ND`, flotantes). Aqu� ocurre la mezcla de tipos. |
| EDA (`X_NUMERICAS`) | DBO, DQO, SST, COLI_FEC, E_COLI, ENTEROC, OD_PORC, OD_PORC_SUP | Media, mediana, outliers, correlaciones y Pipeline del Lab 1. **No** entran a K-means. |
| Geogr�ficas (`COLS_GEO`) | `LONGITUD`, `LATITUD` | **X del agrupamiento.** Resuelven la parte de ubicaci�n del enunciado. No se imputan. |
| Calidad (`Y`) | `SEMAFORO` | **Validaci�n** de la relaci�n (Verde / Amarillo / Rojo). No se usa para armar los clusters. |
| Contexto | `GRUPO`, `ESTADO`, `CUMPLE_CON_*` | `GRUPO` sirve para comprobar nulos por tipo de agua (COSTERO, L�TICO, L�NTICO). No se modelan. |

**Qu� resuelve el enunciado:** `LONGITUD` y `LATITUD` (K-means) m�s `SEMAFORO` (cruce posterior). `X_NUMERICAS` apoyan el EDA; no son las variables del agrupamiento geogr�fico.

---

## 3. Limpieza de datos

### 3.1 El archivo CSV no se modifica

`pd.read_csv` carga una copia en memoria (`df_crudo`). La limpieza crea otro DataFrame (`df`). El archivo en disco o en Drive no se sobrescribe.

- `df_crudo`: estado original (`head`, nulos antes, celdas `<2`).
- `df`: versi�n lista para EDA y K-means.

### 3.2 B�squeda de nulos

Se cuenta `isna().sum()` por columna. Se muestran solo las que tienen al menos un nulo y su porcentaje. El mismo conteo se hace **antes** y **despu�s** de convertir.

### 3.3 Filas que no son un sitio

Se eliminan registros sin `CLAVE` o con `CLAVE` vac�a (filas en blanco al final del CSV).

### 3.4 Datos mal escritos y sustituci�n (datos censurados)

Las columnas de laboratorio mezclan texto y n�mero: `6`, `4.26`, `<2`, `ND`. La t�cnica es **sustituci�n de datos censurados** (*LOD substitution*).

| Caso en el CSV | Decisi�n | T�cnica | Justificaci�n |
|---|---|---|---|
| `ND` o celda vac�a | `NaN` | No imputar | No se midi�. No se inventa un n�mero. |
| `<2`, `<10`, `<3` | LD / 2 | Sustituci�n LOD/2 (censura izquierda) | Convenci�n de EDA. No es la medici�n real. |
| `>100` | El n�mero del l�mite | Sustituci�n por el l�mite (censura derecha) | Permite EDA. Se pierde que era �mayor que�. |
| `6`, `4.26` | Se deja como `float` | Conversi�n num�rica | Ya es una medici�n. |

Ejemplo: en `DBO_mg/L`, `<2` pasa a `1.0`. Un `4.26` se queda. LOD/2 **no** se aplica a toda la base: solo a celdas de `COLS_LAB` que empiezan con `<`. Sem�foro, estado y `CUMPLE_CON_*` no se convierten as�. Lat/lon solo se pasan a `float`.

### 3.5 Otra f�rmula: LOD / ?2

Tambi�n existe **LOD/?2** (~0.707 � LD), usada a veces si se supone lognormalidad. En este proyecto se aplica **LOD/2** por simplicidad: `<2` ? `1`. Ninguna f�rmula es la concentraci�n verdadera. No se implement� LOD/?2.

La sustituci�n es v�lida como convenci�n de limpieza para el curso. No es Kaplan�Meier ni ROS; el enunciado no los pide.

---

## 4. Eliminaci�n de nulos (selectiva)

No se borra toda fila con alg�n NaN ni toda columna con un faltante.

| Nulo | Decisi�n | Motivo |
|---|---|---|
| `LATITUD` o `LONGITUD` faltante | Eliminar la fila | K-means no puede agrupar sin coordenadas. **No se imputan.** |
| `SEMAFORO` faltante | Eliminar la fila | Sin calidad no se valida la relaci�n. **No se imputa.** |
| DBO, DQO, enterococos, ox�geno, etc. | Dejar `NaN` | Ese `GRUPO` puede no medir ese par�metro. Se comprueba despu�s. |

### 4.1 Nulos por `GRUPO`

Despu�s de convertir se calcula el % de nulos de `X_NUMERICAS` por COSTERO / L�TICO / L�NTICO. En estos datos: en COSTERO falta DBO en ~91 % de sitios; en L�TICO y L�NTICO faltan enterococos en ~100 % y 99 %. En COSTERO los enterococos s� se midieron. No es un error de tipeo: no todos los grupos se eval�an con los mismos par�metros.

---

## 5. Relaci�n con el Laboratorio 1

El laboratorio lista cuatro formas de tratar nulos. No se usan las cuatro.

| Opci�n del Lab 1 | �Se us�? | Decisi�n del equipo |
|---|---|---|
| Eliminar toda fila con al menos un nulo | No | Se perder�an costeros (sin DBO) y muchos r�os (sin enterococos). |
| Eliminar toda columna con al menos un nulo | No | Se ir�an DBO, DQO, coliformes y ox�geno. |
| Imputar con media / mediana / constante | Solo en el Pipeline de calidad | Mediana + `MinMaxScaler`. Esa matriz **no** entra a K-means. |
| Imputar y agregar columna flag | No | El enunciado no lo pide. |

Lat/lon se escalan aparte con MinMax **sin imputar**, para que un eje no domine. La inercia **no** son kil�metros.

Se copia el patr�n del Lab 1. Se cambia la herramienta: el lab predice un n�mero; este proyecto agrupa por ubicaci�n y compara calidad.

---

## 6. C�mo se resuelve el enunciado (dos pasos)

La calidad **no** entra a K-means. Si el sem�foro o el DBO entraran al `fit`, el cruce posterior ser�a circular.

### 6.1 Paso 1. Agrupar solo por ubicaci�n

K-means recibe �nicamente `LONGITUD` y `LATITUD` escaladas. Se prueban *k* = 2�10. El *k* del mapa se elige por el **codo**; se reporta silueta. *k* = 3 es solo comparaci�n: tres colores no implican tres regiones.

Resultado: etiqueta `cluster` y centroides. Cada cluster es una **regi�n**, no �agua buena/mala�.

### 6.2 Paso 2. Validar con el sem�foro

`crosstab(cluster, SEMAFORO)` en conteos y en % dentro de cada cluster.

| Lo que se observa | Interpretaci�n |
|---|---|
| Un cluster muy rojo y otro muy verde | Hay patr�n geogr�fico de calidad. |
| Porcentajes parecidos en todos los clusters | La calidad no se explica solo con lat/lon. |

Son **diferencias observadas**, no una prueba estad�stica ni un umbral fijo.

Dos mapas de los mismos puntos: uno por cluster (ubicaci�n) y otro por sem�foro (calidad). Si se parecen, van juntas; si el sem�foro est� salpicado, lat/lon no bastan.

---

## 7. Cierre

Se ajust� K-means para agrupar por coordenadas. **No** se entren� un predictor. La relaci�n calidad�ubicaci�n se obtiene **despu�s**, al cruzar cada regi�n con el sem�foro.

> K-means no clasifica calidad; clasifica sitios por coordenadas. Despu�s comparamos el sem�foro dentro de cada regi�n. Si los porcentajes de verde y rojo cambian entre clusters, la calidad est� ligada a la ubicaci�n. Si no cambian, latitud y longitud no bastan para explicar la calidad del agua.

El CSV original permanece intacto. Las mediciones de laboratorio quedaron num�ricas con LOD/2 (y el l�mite en valores `>LD`). Los nulos de calidad se interpretaron con `GRUPO`. El Pipeline del Laboratorio 1 qued� como preparaci�n de calidad y no se us� para agrupar.

---

## Archivos del proyecto

| Ruta | Contenido |
|---|---|
| `DataSets/` | CSV de sitios y escalas CONAGUA 2020 |
| `solucion_proyecto_final/proyectofinal.py` | Script local (gr�ficas en `graficas/`) |
| `solucion_proyecto_final/Proyecto_Final_Aguas_Superficiales_Colab.ipynb` | Notebook para Google Colab |
| `decisiones _grupo/Informe_decisiones_proyecto_final.docx` | Este mismo informe en Word |
| `decisiones _grupo/decisiones _limpieza_datos.docx` | Bit�cora corta de limpieza |

