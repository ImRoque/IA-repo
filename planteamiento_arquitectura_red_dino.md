# Red Neuronal para DINO CRASH

## Perceptron Simple

Un perceptrón simple es una sola neurona que solo puede tomar decisiones trazando una línea recta para separar los datos (problemas linealmente separables). 

En el juego del dinosaurio, esto falla por completo por dos razones simples:

1. **El dilema de saltar (Problema tipo XOR):**
   - Si viene un **cactus**: saltar te salva (`jump=1` -> vivo), no saltar te mata (`jump=0` -> muerte).
   - Si viene un **pájaro alto**: saltar te mata porque chocas con él (`jump=1` -> muerte), quedarse en el suelo te salva (`jump=0` -> vivo).
   - **Conclusión:** La acción de "saltar" a veces es buena y a veces es mala dependiendo del obstáculo. No existe ninguna línea recta que pueda separar los casos de muerte de los de supervivencia cuando las reglas se cruzan así.

2. **La distancia depende de la velocidad:**
   - Estar a 30 px de un obstáculo a velocidad baja (6.0) te da tiempo de reaccionar.
   - Estar a 30 px a velocidad alta (13.0) es muerte segura.
   - El peligro real viene de combinar ambas cosas (tiempo = distancia / velocidad). Un perceptrón simple solo sabe sumar variables (`w1*dist + w2*vel`), no sabe combinarlas de forma proporcional.


## Perceptron Multicapa

Un perceptrón multicapa tiene capas intermedias (capas ocultas) con funciones no lineales (como ReLU).

1. **Aprende reglas combinadas y curvas:**
   - La primera capa aprende condiciones básicas: *"está cerca"*, *"viene pájaro"*, *"va muy rápido"*.
   - La siguiente capa combina esas condiciones: *"si viene pájaro Y salto -> peligro"*, o *"si viene cactus Y NO salto -> peligro"*.
   - Esto le permite curvar su decisión y resolver cruces como el del cactus y el pájaro.

2. **Aprende la zona de choque:**
   - El salto del dinosaurio es una parábola y los obstáculos son rectángulos.
   - El MLP puede aprender la "zona de impacto" real combinando `dist_obstacle`, `speed`, `dino_height` y `jump`, prediciendo con precisión si habrá choque en el siguiente frame.


## Conclusion 

- **Perceptrón simple:** Solo traza una línea fija. Como en el juego la misma acción (saltar) a veces te salva y a veces te mata según el obstáculo, una línea recta no sirve.
- **Perceptrón multicapa:** Al tener capas ocultas, puede aprender reglas compuestas (*"depende de..."*) y combinaciones entre velocidad, distancia y tipo de obstáculo.


## Arquitectura del Multicapa (MLP)



### 1. Capa de Entrada (8 datos del juego)
Recibe la "foto" del momento exacto (valores escalados entre 0 y 1):
- `dist_obstacle`: qué tan lejos está el obstáculo.
- `speed`: qué tan rápido avanza el juego.
- `dino_height`: altura del dinosaurio en ese instante.
- `jump`: si está saltando (1) o en el piso (0).
- `is_ducking`: si está agachado (1) o no (0).
- `is_cactus_small`: si el obstáculo es cactus chico (1 o 0).
- `is_cactus_large`: si el obstáculo es cactus grande (1 o 0).
- `is_bird`: si el obstáculo es un pájaro (1 o 0).



### 2. Capas Ocultas (Donde se aprende la lógica)
- **Capa Oculta 1**
  Detecta situaciones básicas por separado (ej. "obstáculo muy cercano", "velocidad peligrosa", "dino en el aire").
- **Capa Oculta 2**
  Cruza esas señales para armar reglas compuestas (ej. "cactus cerca" + "dino en el piso" = alerta de choque).


### 3. Capa de Salida (El veredicto)
- **1 sola neurona con activación Sigmoide:**
  Da una salida entre **0.0 y 1.0** que representa la probabilidad de morir en el siguiente frame.
  - Cercano a 0 = No hay peligro.
  - Cercano a 1 = Muerte inminente en el siguiente frame.

### 4. Puntos a tener en cuenta
- **Desbalance:** Como solo el 0.4% de los frames son muertes, se debe penalizar mucho más a la red cuando no avisa de una muerte (*class_weight* o función de pérdida con peso a la clase 1). Si no, la red aprenderá a decir siempre 0.
- **División train/test:** Se deben separar por partidas completas, nunca mezclando frames de una misma partida en entrenamiento y prueba.

