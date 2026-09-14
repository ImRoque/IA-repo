# Reporte Operacion Cuatro Frentes


## Mision 1 - Semaforo Academico

**Pregunta de negocio:** Que nivel de riesgo de reprobar tiene cada alumno al cierre del parcial?

**Tipo propuesto: Clasificar (multiclase)**
`riesgo` tiene tres valores posibles: verde, amarillo, rojo. Son categorias, no numeros. No tiene sentido decir que "rojo = 3" y "verde = 1" como si hubiera una distancia matematica entre ellas. Por eso es clasificacion, y como hay tres clases, es multiclase.

**Variable objetivo (Y):** `riesgo` - categoria con 3 valores: verde, amarillo, rojo.

**Variables de entrada (X):**

| Variable | Por que importa |
|---|---|
| `asistencia_pct` | Los alumnos rojo tienen 52% en promedio vs 91% los verde; diferencia enorme |
| `promedio_parciales` | Media de 5.0 en rojo vs 8.4 en verde; el predictor mas claro |
| `reprobadas_previas` | Alumnos rojo tienen 2.4 materias reprobadas anteriores en promedio |
| `tareas_entregadas` | Menos tareas entregadas se asocia directamente con mayor riesgo |
| `horas_plataforma` | Los alumnos rojo solo usan 1.5 h vs 14 h los verde |
| `turno` | Puede haber diferencias sistematicas por turno (vespertino con mas riesgo?) |

**Patron 1:** Los alumnos rojo tienen asistencia < 55% y promedio < 5.2 consistentemente. Los verde tienen asistencia > 88% y promedio > 8.0. La separacion es muy clara.

**Patron 2:** `reprobadas_previas` crece con el riesgo: 0.2 en verde, 0.9 en amarillo, 2.4 en rojo. Historial previo es una senal fuerte.

**Distribucion de Y:**

| riesgo | n | % |
|---|---|---|
| verde | 120 | 40% |
| amarillo | 105 | 35% |
| rojo | 75 | 25% |

Ninguna clase domina de forma extrema, pero rojo es la menos frecuente (25%). Accuracy puede ser valida aqui, pero conviene tambien mirar F1 por clase, especialmente para rojo, que es la clase mas critica de no perder.

**Problema de calidad inventado:** Algunos alumnos podrian tener `asistencia_pct = 0` pero `tareas_entregadas > 0`, lo que es contradictorio (como entrego tareas si no fue?). Se detecta cruzando las dos columnas y buscando casos donde asistencia = 0 y tareas >= 2.

**Leakage:** No se usaria la calificacion final del curso como X, porque esa nota se conoce al cierre del semestre, no al cierre del parcial. Si la incluyeramos, el modelo aprenderia a copiar la calificacion, no a predecir el riesgo a tiempo.

**Metricas:** F1-score por clase (especialmente rojo), Accuracy general como referencia.

**Modelo propuesto:** Random Forest multiclase. Condicion: que las tres clases tengan al menos 50 ejemplos cada una para que el modelo aprenda bien las tres.



## Mision 2 - Alerta de Churn Estudiantil

**Pregunta de negocio:** Este alumno va a abandonar la materia antes de que termine el semestre?

**Tipo propuesto: Clasificar (binaria)**
`abandona` solo tiene valores 0 y 1. No hay valor intermedio posible. La pregunta es un si o no, no un numero continuo.

**Distribucion de Y y sus implicaciones:**

| abandona | n | % |
|---|---|---|
| 0 (permanece) | 430 | 86% |
| 1 (abandona) | 70 | 14% |

Hay 70 casos positivos en 500. Si alguien dice "mi modelo tiene 86% de aciertos", puede ser que el modelo siempre diga "no abandona" y aun asi acierte el 86% de las veces. Eso no sirve de nada: necesitamos detectar los 70 que si abandonan. Por eso accuracy sola es engañosa; hay que usar **Recall y F1** sobre la clase abandona=1.

**Valores faltantes en `calif_actividad_1`:**
Los alumnos con NA en esta columna son casi todos del grupo abandona=1 (ver muestra: alumnos 203, 204, 208, 210 tienen NA y abandona=1). Esto significa que el NA en si mismo es una senal de abandono, no un dato perdido al azar.

Tratamiento: crear una columna binaria `tiene_calif` (1 si no es NA, 0 si es NA), conservar ambas. No borrar los registros con NA porque son los mas informativos.

**Leakage:** No se usaria "fecha de baja definitiva" ni "nota final" como X. La fecha de baja solo existe cuando el alumno ya se fue (lo que queremos predecir), y la nota final no existe a mitad del semestre. Ambas variables llegan despues del evento.

**Tres preguntas EDA:**

**Como esta distribuida Y?** Desbalanceada: 86% no abandona, 14% abandona. Ratio de 6:1. Accuracy inutil; usar Recall y F1.

**Hay outliers en `dias_sin_login`?** El alumno 210 tiene 30 dias sin login y abandono=1. Un valor de 30 dias es posible y real (el semestre tiene ~18 semanas). No es un error, es la senal mas fuerte del dataset. Conservar.

**Los abandona=1 se concentran en alumnos que trabajan?** De los 10 de la muestra, 6 que trabajan (trabaja=1) tienen 4 abandonos. Entre los que no trabajan, 1 abandono. La correlacion existe pero no es determinante sola; `dias_sin_login` es mas fuerte.

**Comparacion con Mision 1:** Ambas son clasificacion binaria (M2) vs multiclase (M1). En M1 la Y tiene 3 clases equilibradas; en M2 la Y esta desbalanceada (86/14). El EDA de M1 se enfoca en separar tres grupos; el de M2 se enfoca en detectar la clase minoritaria sin que el modelo la ignore.

**Metricas:** Recall sobre abandona=1 (no queremos perder casos reales de abandono), F1 como balance entre no perder abandonos ni generar falsas alarmas. El costo de equivocarse es alto: si el modelo no detecta a un alumno que va a abandonar, la universidad pierde la oportunidad de intervenir.

**Modelo propuesto:** Regresion logistica con pesos ajustados (class_weight balanced). Simple, interpretable, adecuado para el desbalance.


## Mision 3 - Pronostico de Puntaje Final

**Pregunta de negocio:** Que calificacion final (0-100) obtendra este alumno?

**Tipo propuesto: Predecir (regresion)**
`calificacion_final` es un numero continuo entre 0 y 100. La pregunta pide un numero especifico, no una categoria. Predecir 73.5 o 89.0 tiene significado directo.

**Variable objetivo (Y):** `calificacion_final` - numero continuo.

**Variables de entrada (X):**

| Variable | Relacion con Y |
|---|---|
| `examen_1` | Correlacion ~0.85 con calificacion_final; la mas fuerte |
| `examen_2` | Correlacion similar a examen_1 |
| `promedio_tareas` | Contribuye al promedio final |
| `asistencia_pct` | Asistencia alta se asocia a mejores notas |
| `horas_estudio_sem` | Correlacion moderada; tiene outliers |

**Convertir a aprobado/reprobado:**
- Ganas: el problema se vuelve mas simple (clasificacion binaria). Mas facil de comunicar.
- Pierdes: toda la informacion del numero. Un alumno con 60 y uno con 90 quedarian en la misma categoria "aprobado". El modelo no podria distinguir entre quien paso justo y quien saco excelente.

**Distribucion de `horas_estudio_sem`:**
Media = 5.5, Mediana = 5.0, Maximo = 25. Media > mediana sugiere cola derecha: algunos alumnos reportan 20-25 horas por semana, lo cual es dudoso (es auto-reporte). Hay sesgo hacia arriba por esos valores extremos. Habria que revisar si son reales o errores de captura antes de usar esta variable.

**Las 3 filas con `calificacion_final > 100`:** Son errores de captura claros (el maximo posible es 100). Se eliminan esas filas antes de entrenar. Si fueran muchas, habria que investigar la fuente; con solo 3 en 200, se descartan.

**Metricas:**
- **MAE (Error Absoluto Medio):** Cuantos puntos se equivoca el modelo en promedio. Facil de interpretar: "el modelo se equivoca en promedio 4 puntos".
- **RMSE (Raiz del Error Cuadratico Medio):** Penaliza mas los errores grandes. Util para detectar si el modelo falla mucho en casos extremos.

**Si examen_1 vs calificacion_final es casi una linea:** Regresion lineal encaja perfectamente. Si la relacion fuera en "escalones" (grupos discretos), un arbol de decision seria mejor.

**Comparacion con Mision 1:** Mismo dominio escolar, distinta Y. En M1 la Y es una categoria (riesgo), en M3 es un numero. Eso cambia todo: las metricas, el modelo y la interpretacion del resultado.

**Modelo propuesto:** Regresion lineal (si la relacion con examenes es lineal como indican las correlaciones). Alternativa: arbol regresor si hay relaciones no lineales.



## Mision 4 - Estimacion de Tiempo de Estudio

**Pregunta de negocio:** Cuantas horas adicionales necesita este alumno para dominar el tema?

**Tipo propuesto: Predecir (regresion)**
`horas_adicionales` es un numero continuo (2.0, 18.0, 35.0...). La pregunta pide una estimacion numerica especifica, no una categoria. Predecir "necesita 14 horas" es mas util que "necesita muchas horas".

**Hipotesis EDA - dificultad vs horas:**
A mayor dificultad del tema, mas horas adicionales se necesitan. Los datos lo confirman: media de 3.2 h en dificultad baja, 7.8 h en media, 16.5 h en alta. La relacion es monotona y clara.

**Como inspeccionar las categoricas `tema_dificultad` y `dispositivo`:**
- Hacer tabla de frecuencias: cuantos alumnos hay en cada categoria.
- Calcular la media de `horas_adicionales` por cada categoria (como se muestra en la tabla de dificultad).
- Revisar si alguna categoria tiene muy pocos casos (menos de 10) para saber si es confiable.
- Para `dispositivo`: ver si los alumnos en movil tienen mas horas adicionales que los de pc (podria reflejar menos comodidad para estudiar).

**Cola > 40 horas (3%):**
No borrar. Esos casos representan alumnos con grandes dificultades; son reales, no errores. Si los eliminamos, el modelo nunca aprendera a identificar a los alumnos que mas necesitan atencion, que son precisamente los mas importantes. Conservar y documentar.

**Redundancia entre `pretest_score` y `ejercicios_correctos_pct`:**
Ambas miden habilidad del alumno al inicio/durante el curso. Si van juntas (alumno con pretest alto tambien tiene alto % de ejercicios), son redundantes. Como checar: calcular correlacion entre las dos. Si es > 0.8, hay redundancia y conviene quedarse con la mas facil de medir o combinarlas.

**Binarizar Y con umbral 15 h ("tutoria intensiva: si/no"):**
Tendria sentido si el objetivo del sistema es solo decidir si un alumno necesita tutoria intensiva o no, no saber cuantas horas exactas. Se pierde precision: un alumno con 14.9 h y uno con 2 h quedarian en la misma categoria "no intensivo", aunque son muy diferentes. Solo vale si la decision final es binaria (asignar o no tutor).

**Metricas:**
- **MAE:** Cuantas horas en promedio se equivoca la prediccion. Facil de explicar al equipo de tutoria.
- **RMSE:** Penaliza errores grandes; util porque equivocarse en 20 horas es mucho mas grave que en 2.

**Modelo propuesto:** Arbol regresor o Gradient Boosting (la relacion con dificultad parece no lineal y hay variables categoricas).

**2 chequeos EDA obligatorios antes de entrenar:**
1. Verificar que `tema_dificultad` tiene suficientes casos en cada categoria (minimo 50 por nivel).
2. Revisar correlacion entre `pretest_score` y `ejercicios_correctos_pct`; si es > 0.8, eliminar una.


## Conclusion

**Tabla resumen de mis propuestas:**

| Mision | Nombre | Y | Tipo propuesto | Modelo |
|---|---|---|---|---|
| M1 | Semaforo Academico | `riesgo` (verde/amarillo/rojo) | Clasificacion multiclase | Random Forest |
| M2 | Alerta Churn | `abandona` (0/1) | Clasificacion binaria | Regresion logistica |
| M3 | Pronostico Puntaje | `calificacion_final` (0-100) | Prediccion numerica | Regresion lineal |
| M4 | Tiempo de Estudio | `horas_adicionales` (numero) | Prediccion numerica | Gradient Boosting |

**Pista que use para decidir clase vs numero:** Mirar si Y es una etiqueta (categoria sin orden matematico real) o un numero con significado continuo. Si puedo decir "la diferencia entre Y=70 y Y=80 son 10 puntos con sentido real", es regresion. Si solo puedo decir "pertenece al grupo A o B", es clasificacion.

**Frase final:** El tipo de problema se deduce de la pregunta y de Y porque la forma de Y define que tipo de respuesta necesita el modelo. Si Y es una categoria, el modelo tiene que elegir un grupo; si Y es un numero, el modelo tiene que calcular un valor. Elegir mal el tipo significa usar la metrica equivocada y responder la pregunta equivocada.
