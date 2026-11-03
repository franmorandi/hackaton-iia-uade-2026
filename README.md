# M15 · ¿Este jugador vuelve la semana que viene?

**Integrantes:** Mateo Lioi · Franco Morandi Roch
**IA declarada:** Gemini
**Pista:** Reglas
**Publicación en LinkedIn:**
Mateo Lioi: 
Franco Morandi Roch: 

---

## 1. Formulación y PEAS

**La pregunta.** Para cada caso, ¿el jugador NO vuelve a jugar la semana siguiente? SÍ significa: el jugador abandona el juego.

| PEAS | Su sistema |
|---|---|
| Medida de rendimiento | Sensibilidad (Recall) en la detección de abandonos reales y exactitud global de clasificación. |
| Entorno | Base de datos de telemetría semanal del videojuego y sistema de gestión de Live Ops. |
| Actuadores | Otorgamiento automático de contenido gratuito / regalos de fidelización a los jugadores en riesgo de abandono. |
| Sensores | Métricas semanales del jugador: sesiones semanales, días trabado, amigos activos y minutos jugados. |

**Objetivo y métrica.** 
* **Objetivo del sistema:** Clasificar correctamente a cada jugador en riesgo de abandono (emitir la respuesta binaria SÍ o NO) para que el equipo de Live Ops intervenga a tiempo. El sistema no calcula cuántos minutos jugará la persona, sino que predice su estado de retención.
* **Métrica con la que afirmamos que el sistema funciona:** La Sensibilidad (Recall). El **83,33%** obtenido en entrenamiento es el número técnico con el que demostramos que el sistema funciona para detectar abandonos a tiempo.

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

**Entrenamiento contra prueba.** Presenta una **caída drástica y significativa de la sensibilidad** (pasa del 83,33% al 50,00%). Esto ocurre porque el dataset de prueba consta de solo 6 casos y contiene 2 casos reales de abandono (15 y 19). Al no detectar el Caso 19, la sensibilidad cae 33,33 puntos porcentuales debido a la reducida escala de la muestra. Esto ocurre porque el dataset de prueba es muy reducido (6 casos) y contiene un caso límite (Caso 19: 6 sesiones con 13 días trabado), donde el nivel de frustración provocó el abandono a pesar de haber ingresado 6 veces. No se observa sobreajuste (overfitting) severo, manteniendo una especificidad y precisión perfectas.

## 4. El error más caro

El error más caro es el **Falso Negativo (dice NO y era SÍ)**.

* **Por qué:** Si el modelo predice que un jugador se queda cuando en realidad abandona, no se le envía contenido de fidelización y el usuario se pierde de forma definitiva. Un Falso Positivo solo implica el costo marginal de regalar contenido a alguien que continuaba jugando.
* **Análisis de la regla de sesiones y fallos (Casos 11 y 19):**
  * **Caso 11 (Entrenamiento):** `Sesiones = 7`, `Días trabado = 11`, Real: **SÍ**.
  * **Caso 19 (Prueba):** `Sesiones = 6`, `Días trabado = 13`, Real: **SÍ**.
  * **Causa del fallo:** La Regla 1 exige `Sesiones <= 5`. Al ingresar 7 y 6 veces, el sistema predijo NO (Falso Negativo).
  * **Qué probamos y por qué no alcanzó:** Evaluamos ampliar la regla de sesiones a `Sesiones <= 7` para capturar a los casos 11 y 19. Si bien esto elevaba la sensibilidad, **provocaba Falsos Positivos severos en el Caso 2 (`Sesiones = 7`, `Días trabado = 9`, Real: NO) y Caso 10 (`Sesiones = 9`, `Días trabado = 11`, Real: NO)**. Decidimos mantener la regla estricta en `Sesiones <= 5` para no destruir la precisión perfecta (100%) regalando contenido a jugadores activos[cite: 2].

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
| ¿Quién lo midió? | Franco Morandi Roch y Mateo Lioi mediante evaluación directa sobre la partición fija dada por la cátedra (casos 1-14 para entrenamiento y 15-20 para prueba). |

## 6. Riesgo: el camino completo

Datos → Modelo → Decisión → Consecuencia 

Escenario de riesgo: Si se introduce una actualización en el juego con un nivel defectuoso (bug) que impide avanzar a todos los usuarios, la variable Días trabado se disparará de forma artificial para toda la base de jugadores.   

Consecuencia: El modelo clasificará masivamente a los jugadores como SÍ (en riesgo de abandono) y el sistema de Live Ops regalará automáticamente contenido de pago a usuarios que no tenían intención real de irse, provocando una pérdida severa de ingresos por venta de contenido.

## 7. Bitácora

**Uso de la IA**

| # | Qué le pedimos | Qué aceptamos | Qué rechazamos y por qué |
|---|---|---|---|
| 1 | Formular un sistema de reglas usando la variable Minutos jugados. | Aceptamos evaluar la frustración mediante la variable Días trabado. | Rechazamos incorporar Minutos jugados como regla principal porque castigaba a usuarios nuevos activos con pocas horas totales acumuladas |
| 2 | Proponer un cambio de umbral a Sesiones <= 7 para capturar los casos 11 y 19. | Aceptamos testear el impacto del nuevo umbral en el dataset | Rechazamos cambiar a Sesiones <= 7 porque provocaba Falsos Positivos en los casos 2 y 10, arruinando la precisión perfecta (100%). |
| 3 | Analizar la brecha de rendimiento entre entrenamiento y prueba. | Aceptamos los cálculos exactos de las matrices de confusión. | Rechazamos calificar la baja de sensibilidad del 83,33% al 50,00% como "ligera", redefiniéndola como una caída drástica explicada por la escala reducida del set de prueba (N = 6). |

**Corridas contra el evaluador**

| Corrida | Nota | Agujero que encontró | Qué cambiamos |
|---|---|---|---|
| 1 | 7,5 | "Definiste el objetivo como sostener los minutos jugados. ¿Ese número es lo que querés responder con el sistema, o es uno que lo acompaña?" | El cambio principal fue en el objetivo y la métrica del sistema, ya que era el causante del agujero principal planteado por el bot en su devolución. Además, modificamos y corregimos la parte que mencionaba que medimos mediante validación cruzada, dado que era incorrecta y no había sido realizada de esa manera. Por lo tanto, también modificamos la bitácora de la IA, ya que esta parte fue provista por la misma y aceptada (por ende, no rechazada) por nosotros.  |
| 2 | 7,5 | "En la sección 1 y en la sección 4 seguís diciendo que el éxito se mide con los minutos jugados. ¿Con qué número decís que tu sistema funciona?" | Cambiamos la medida del PEAS por Sensibilidad (83,33%); eliminamos minutos como rendimiento en Sec 4; cambiamos «ligera» por «drástica y significativa»; explicamos la regla de Sesiones <= 5 vs 7 para casos 11 y 19; fundamentamos rechazos y registramos Corrida 2. Además, modificamos la parte del método de medición de la sección 5 (no habíamos guardado el cambio la vez anterior). |
| 3 | 8 |"Repasá tu argumento para no subir a 7 sesiones. ¿Qué pasa con el caso 10 con esa regla? Contá cuántos falsos positivos y falsos negativos tendrías en total, y decime cuál de los dos dijiste que es más caro." | - |
