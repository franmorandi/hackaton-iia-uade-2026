# M15 · ¿Este jugador vuelve la semana que viene?

**Integrantes:** Mateo Lioi · Franco Morandi Roch
**IA declarada:** Gemini
**Pista:** Reglas
**Publicación en LinkedIn:** <link>

---

## 1. Formulación y PEAS

**La pregunta.** Para cada caso, ¿el jugador NO vuelve a jugar la semana siguiente? SÍ significa: el jugador abandona el juego.

| PEAS | Su sistema |
|---|---|
| Medida de rendimiento | Tasa de retención de jugadores y minutos jugados totales preservados (minimizando el abandono no detectado). |
| Entorno | Base de datos de telemetría semanal del videojuego y sistema de gestión de Live Ops. |
| Actuadores | Otorgamiento automático de contenido gratuito / regalos de fidelización a los jugadores en riesgo de abandono. |
| Sensores | Métricas semanales del jugador: sesiones semanales, días trabado, amigos activos y minutos jugados. |

**Objetivo y métrica.** El objetivo central es evitar que los jugadores abandonen para sostener los minutos jugados totales. La métrica crítica es la sensibilidad (recall), ya que mide la capacidad de detectar a todos los jugadores que están por irse (reducir Falsos Negativos). No son exactamente lo mismo: el objetivo es de negocio (retención y minutos) y la métrica es de evaluación del modelo (sensibilidad de clasificación).

## 2. El sistema

**Cómo decide.** El sistema utiliza un conjunto de reglas lógicas de decisión (pista de reglas):
REGLA 1: Si Sesiones <= 5 Y Días trabado >= 7 → Predice SÍ (el jugador NO vuelve).   
REGLA 2: En cualquier otro caso → Predice NO (el jugador vuelve). 

**Variables que usa y por qué.** 
Variables usadas: Sesiones (evalúa la frecuencia semanal de ingreso) y Días trabado (mide el nivel de frustración por falta de progreso en el juego).   
Variables descartadas: Amigos activos y Minutos jugados. Se descartaron porque la combinación de baja frecuencia y alta frustración demostró ser el factor determinante de abandono, independientemente del historial previo de minutos o red social.   

**Cómo se corre en vivo.** Para evaluar un jugador nuevo en la semana actual:
1. Tomar el valor semanal de Sesiones y Días trabado del tablero de analítica.   
2. Verificar si cumple simultáneamente Sesiones <= 5 y Días trabado >= 7.   
3. Si cumple ambas condiciones, el sistema de Live Ops le asigna la etiqueta SÍ y activa el envío del paquete de contenido gratuito.  

**Dos casos de ejemplo.**

| Caso inventado | Valores | Salida del sistema |
|---|---|---|
| A | Sesiones: 2, Días trabado: 10, Amigos: 1, Minutos: 180 | SÍ (Abandona, se envía contenido) |
| B | Sesiones: 8, Días trabado: 2, Amigos: 5, Minutos: 600 | NO (Permanece, no se envía regalo) |

## 3. Medición

**Entrenamiento (casos 1–14)**

| | Predice SÍ | Predice NO |
|---|---|---|
| **Real SÍ** | VP = 5 (Casos 1, 5, 7, 8, 13) | FN = 1 (Caso 11) |
| **Real NO** | FP = 0 | VN = 8 (Casos 2, 3, 4, 6, 9, 10, 12, 14) |

**Prueba (casos 15–20)**

| | Predice SÍ | Predice NO |
|---|---|---|
| **Real SÍ** | VP = 1 (Caso 15) | FN = 1 (Caso 19) |
| **Real NO** | FP = 0 | VN = 4 (Casos 16, 17, 18, 20) |

| Métrica | Entrenamiento | Prueba |
|---|---|---|
| Exactitud | 92.86% (13/14) | 83.33% (5/6) |
| Precisión | 100.00% (5/5) | 100.00% (1/1) |
| Sensibilidad | 83.33% (5/6) | 50.00% (1/2) |
| Especificidad | 100.00% (8/8) | 100.00% (4/4) |

**Entrenamiento contra prueba.** Existe una ligera disminución de la exactitud y sensibilidad en la fase de prueba. Esto ocurre porque el dataset de prueba es muy reducido (6 casos) y contiene un caso límite (Caso 19: 6 sesiones con 13 días trabado), donde el nivel de frustración provocó el abandono a pesar de haber ingresado 6 veces. No se observa sobreajuste (overfitting) severo, manteniendo una especificidad y precisión perfectas.

## 4. El error más caro

El error más caro es el Falso Negativo (dice NO y era SÍ).   
Por qué: El estudio mide su éxito principal a través de los minutos jugados totales. Si cometemos un Falso Negativo, no detectamos a un jugador que está a punto de irse y no se le ofrece contenido nuevo, por lo que abandona la plataforma definitivamente y se pierden todos sus minutos jugados futuros. En cambio, un Falso Positivo sólo implica regalar contenido a alguien que habría continuado jugando.   
Ajuste realizado: Se fijó el umbral de Días trabado en $\ge 7$ para capturar a los usuarios frustrados a partir de la primera semana de estancamiento. 

## 5. ¿Comparado contra qué?

**Línea base trivial.** Un sistema trivial que prediga siempre NO (asumir que ningún jugador abandona):   
Exactitud trivial (Entrenamiento): 57.14% (8 de 14 correctos).   
Sensibilidad trivial: 0.00% (no detecta a ningún jugador que abandona).   
Nuestro sistema supera ampliamente la línea base al alcanzar una exactitud del 92.86% en entrenamiento y detectar el 83.33% de los abandonos reales.  

| Checklist (Clase 6) | Respuesta |
|---|---|
| ¿Qué datos? | Archivo datos/M15.csv (20 registros de interacción semanal de jugadores). |
| ¿Qué métrica? | Sensibilidad (Recall) como métrica primaria y Exactitud (Accuracy) como secundaria. |
| ¿Comparado contra qué? | Clasificador constante trivial ("Siempre NO"). |
| ¿Quién lo midió? | Franco Morandi Roch y Mateo Lioi mediante validación cruzada entre conjuntos. |

## 6. Riesgo: el camino completo

Datos → Modelo → Decisión → Consecuencia 

Escenario de riesgo: Si se introduce una actualización en el juego con un nivel defectuoso (bug) que impide avanzar a todos los usuarios, la variable Días trabado se disparará de forma artificial para toda la base de jugadores.   

Consecuencia: El modelo clasificará masivamente a los jugadores como SÍ (en riesgo de abandono) y el sistema de Live Ops regalará automáticamente contenido de pago a usuarios que no tenían intención real de irse, provocando una pérdida severa de ingresos por venta de contenido.

## 7. Bitácora

**Uso de la IA**

| # | Qué le pedimos | Qué aceptamos | Qué rechazamos y por qué |
|---|---|---|---|
| 1 | Le pidieron a la IA explorar un sistema de reglas para detectar jugadores en riesgo de abandono priorizando la sensibilidad. Le pidieron calcular las matrices de entrenamiento y prueba para los 20 casos de la ficha M15. | Para esta primera corrida, luego de una revision completa hecha por nosotros, asegurándonos que nada de lo entregado por la IA resulte incoherente, optamos por aceptar la totalidad de lo provisto por la misma. | Dado que aceptamos todo lo que entregó la IA, no se rechazó nada. |

**Corridas contra el evaluador**

| Corrida | Nota | Agujero que encontró | Qué cambiamos |
|---|---|---|---|
| 1 | <> | <> | <> |
