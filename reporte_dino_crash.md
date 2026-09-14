# Reporte Operacion Dino Crash

## 1. Problema y Dataset

### Va a morir el dino en el siguiente frame?

- **Y:** `died_next_frame` - si (1) o no (0). Se construye moviendo `died` un frame hacia adelante.
- **Tipo:** Clasificacion binaria
- **Una fila = un frame (~16 ms)**

**Variables de entrada (X):**

| Variable | Por que importa |
|---|---|
| `dist_obstacle` | Distancia al obstaculo; si es muy poca, hay peligro inmediato |
| `speed` | A mas velocidad, menos tiempo para reaccionar |
| `obstacle_type` | No es igual un cactus que un pajaro |
| `jump` | Si ya esta en el aire no puede volver a saltar |
| `dino_height` | Saber si el salto fue suficiente para pasar el obstaculo |

- **Frecuencia:** Cada frame (~16 ms), porque la muerte pasa en un instante.
- **Minimo de datos:** 500 partidas. Las muertes son rarisimas (0.03% de los frames); con menos no hay suficientes ejemplos.
- **Riesgo:** Si usamos `died` tal cual, el modelo detecta la muerte cuando ya paso. Demasiado tarde.

---

### Cuantos puntos hara esta partida?

- **Y:** `final_score` - numero (ej. 42, 500, 1200).
- **Tipo:** Regresion
- **Una fila = una partida completa**

**Variables de entrada (X):**

| Variable | Por que importa |
|---|---|
| `avg_reaction_lag` | Que tan rapido reacciona el jugador |
| `jump_accuracy` | % de saltos exitosos en los primeros 10 s |
| `max_speed_reached` | La velocidad maxima alcanzada antes de morir |
| `obstacle_density` | Cuantos obstaculos por segundo hubo |
| `early_death` | Murio antes de 5 s? (probablemente fue accidente) |

- **Frecuencia:** Un resumen por partida, no por frame.
- **Minimo de datos:** 2,000 partidas.
- **Riesgo:** Si incluimos el `score` actual como variable, hacemos trampa: en el ultimo frame el score actual *es* el score final.

---

### Que tipo de obstaculo viene despues?

- **Y:** `next_obstacle_type` - uno de cuatro: `none, cactus_small, cactus_large, bird`.
- **Tipo:** Clasificacion de varias categorias
- **Una fila = el momento justo despues de pasar un obstaculo**

**Variables de entrada (X):**

| Variable | Por que importa |
|---|---|
| `score` | A mas puntos, mas probabilidad de pajaros |
| `speed` | La velocidad cambia que obstaculos aparecen |
| `last_obstacle_type` | El juego no suele repetir el mismo tipo dos veces |
| `dist_obstacle` | Que tan lejos esta el siguiente |
| `time_since_last_obstacle` | El ritmo entre obstaculos da pistas |

- **Frecuencia:** Una fila por obstaculo superado, no por frame.
- **Minimo de datos:** 10,000 eventos; el pajaro (11%) necesita suficientes ejemplos.
- **Riesgo:** Si etiquetamos el obstaculo actual en lugar del siguiente, el modelo no anticipa nada.



## 2. Diccionario y Muestra

### Columnas propuestas

| Columna | Tipo | Sirve para P1? |
|---|---|---|
| `session_id` | entero | Si |
| `frame` | entero | Si |
| `time_ms` | entero | Si |
| `score` | entero | No predice peligro inmediato |
| `speed` | decimal | Si, importante |
| `obstacle_type` | categoria | Si, importante |
| `dist_obstacle` | decimal | Si, muy importante |
| `jump` | 0/1 | Si |
| `died` | 0/1 | Necesita transformarse |

**Lo que falta:** `dino_height`, `is_ducking`, `obstacle_height`, `frames_airborne`.

---

### Analisis de la muestra (Sesion 7)

| frame | time_ms | score | speed | obstacle_type | dist_obstacle | jump | died |
|---|---|---|---|---|---|---|---|
| 0 | 0 | 0 | 6.0 | none | 180 | 0 | 0 |
| 40 | 640 | 8 | 6.4 | none | 165 | 0 | 0 |
| 80 | 1280 | 16 | 6.8 | cactus_small | 55 | 1 | 0 |
| 81 | 1296 | 16 | 6.8 | cactus_small | 38 | 1 | 0 |
| **82** | **1312** | **16** | **6.8** | **cactus_small** | **12** | **0** | **1** |

**Patron en `died=1` (frame 82):** `dist_obstacle=12` (muy cerca) + `jump=0` (no esta saltando) = choque. El score se mantuvo en 16 desde el frame 80, lo que confirma que el score **no avisa del peligro**.

**Sobre `died`:** Solo hay un `1` por partida, al final. Para P1 hay que moverlo un frame atras:

died_next = died.shift(-1)




## 3. Checklist EDA

**Clases balanceadas? (P1)**
Con 500 partidas de ~300 frames cada una hay ~150,000 frames y solo 500 muertes -> ratio de 299:1. La **exactitud (accuracy) no sirve**; usamos **F1, Precision y Recall**.

**Distribucion de `speed`?**
Va de 6.0 a 13.0 con media 8.5. Muchas partidas terminan pronto (speed baja), pocas llegan alto. `speed` y `dist_obstacle` juntos importan mas que cada uno solo: a speed=13, una distancia de 30 px da solo 2 frames de margen.

**Outliers?**
Valores negativos en `dist_obstacle`, `speed < 6`, o `jump=1` + `is_ducking=1` al mismo tiempo son errores de captura y deben eliminarse.

**Data leakage?**
Si usamos `score` o `time_ms` para predecir muerte en P1, el modelo aprende detalles especificos de cada sesion (score=16 = muerte en la sesion 7) que no se generalizan a otras.

**Los frames son independientes?**
No. El frame 81 viene del 80. Si mezclamos frames al azar en entrenamiento y prueba, el modelo ve los vecinos de un frame durante el entrenamiento y lo "adivina" facil en la prueba -- es trampa.

**Solucion:** Dividir **por partida completa**:

Partidas 1-350   -> Entrenamiento (70%)
Partidas 351-425 -> Validacion    (15%)
Partidas 426-500 -> Prueba        (15%)


---

## 4. Interpretacion de Estadisticas

| Variable | Media | Mediana | Min | Max |
|---|---|---|---|---|
| `score` | 28 | 18 | 0 | 120 |
| `speed` | 8.5 | 8.2 | 6.0 | 13.0 |
| `dist_obstacle` | 95 | 88 | 5 | 220 |

| Tipo obstaculo | % |
|---|---|
| none | 54% |
| cactus_small | 20% |
| cactus_large | 15% |
| bird | 11% |

**Desbalance en P1:** 50 muertes en 12,000 frames = 0.42% (ratio 239:1). Accuracy inutil aqui -> usar F1.

**`dist_obstacle` como predictor:** La mediana normal es 88 px, las muertes ocurren con dist < 22 px. La diferencia es clara; cualquier modelo lo va a notar.

**Distribucion de `score`:** La media (28) es mayor que la mediana (18) -> hay partidas con scores altos que distorsionan. Para predecir el puntaje final en P2 conviene transformar con `log(score+1)` o usar un arbol.



## 5. Eleccion de Modelo

### Guia rapida

| Lo que muestra el EDA | Modelos que funcionan | Lo que no conviene |
|---|---|---|
| Si/No con desbalance enorme | Random Forest con pesos ajustados, Regresion logistica | Accuracy como metrica |
| Un numero con distribucion desigual | Arbol regresor, Gradient Boosting | Convertirlo en si/no |
| Una de varias categorias | Arbol de decision, Random Forest | Asignarles numeros ordenados |
| Datos con orden en el tiempo | Ventana de frames + clasificador | Mezclar frames al azar |

### Eleccion por escenario

| Escenario | Modelo | Condiciones |
|---|---|---|
| P1 - Muere? | Random Forest con pesos ajustados | >=500 partidas; dividir por sesion |
| P2 - Cuantos puntos? | Gradient Boosting | >=2,000 partidas; transformar score con log |
| P3 - Que obstaculo sigue? | Random Forest multiclase | >=10,000 eventos; one-hot en obstacle_type |

### Regla fija vs modelo

`SI dist_obstacle < 15 Y jump = 0 ENTONCES -> muerte`

- OK: Facil de entender, no necesita datos, no hace trampa.
- No OK: El umbral no cambia con la velocidad; no maneja casos especiales; no aprende sola.

**Recomendacion:** Empezar con la regla fija como referencia. Si el modelo no la supera en F1, la regla es suficiente.



## Conclusion

Pediriamos primero el dataset de P1 (nivel de frame) con: `session_id`, `frame`, `speed`, `obstacle_type`, `dist_obstacle`, `dino_height`, `jump`, `is_ducking` y `died`. Con eso se derivan P2 y P3.

El EDA mostraria el desbalance enorme (accuracy inutil), que `dist_obstacle` es la variable clave, y que los frames no son independientes entre si. Solo despues de eso elegiria el modelo: primero la regla fija, luego regresion logistica, y si hace falta mas, Random Forest.

**El modelo se elige por lo que muestran los datos, no por moda.**

