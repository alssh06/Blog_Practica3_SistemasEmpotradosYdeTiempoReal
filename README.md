## Práctica 3: Controlador de máquina expendedora con Arduino UNO
**Sistemas Empotrados y de Tiempo Real** · Autor: Alejandro López Sánchez

### Montaje
Foto del montaje:

<img width="3000" height="4000" alt="20261122_113431" src="https://github.com/user-attachments/assets/ef8b3fb6-2e3f-46fa-b646-93de9c12d8f3" />

Fritzing del montaje:

<img width="831" height="377" alt="Captura desde 2026-11-22 12-04-55" src="https://github.com/user-attachments/assets/30a3499b-b532-4e79-a96b-db8b442efe58" />

### Componenetes

| Componente | Pin Arduino |
|---|---|
| LED1 (azul) | 12 | 
| LED2 (verde) | 11 | 
| Botón externo | 2 | 
| Joystick eje X | A0 | 
| Joystick eje Y | A1 | 
| Joystick pulsador | A2 | 
| HC-SR04 TRIG | 10 | 
| HC-SR04 ECHO | 9 | 
| DHT11 DATA | A3 |  
| LCD RS, E, D4, D5, D6, D7 | 8, 7, 6, 5, 4, 3 | 

### Guion de funcionamiento del programa
La máquina posee 3 modos principales:
- **ARRANQUE**: LED1 parpadea 3 veces (1 s encendido, 1 s apagado) y el LCD muestra "CARGANDO ...".
- **SERVICIO**: Atiende al cliente: detecta presencia, muestra temperatura, humedad y catálogo, y prepara la bebida, teniendo un método de cancelación.
- **ADMIN**: Permite ver temperatura, humedad, tiempo con la maquina encendida, distancia detectada y modificar precios, el modo se indica con ambos LEDs encendidos.

Y consta del siguiente flujo de acción en su bucle:
1. La máquina **empieza en modo ARRANQUE** y, al terminar los 3 parpadeos, pasa sola a SERVICIO.
2. En SERVICIO muestra **"ESPERANDO CLIENTE"** hasta que el sensor de ultrasonidos detecta una distancia **inferior a 1 m**.
3. Entonces el LCD muestra la **temperatura y la humedad durante 5 s** y después la **lista de productos**.
4. El cliente navega con el **joystick (arriba/abajo)** y confirma con su **pulsador**.
5. Se muestra **"Preparando..."** durante un tiempo **aleatorio entre 4 y 8 s**, mientras el **LED2 aumenta su brillo con PWM** según el progreso.
6. Se muestra **"RETIRE BEBIDA" durante 3 s** y vuelve a "ESPERANDO CLIENTE".
7. El ciclo se repite indefinidamente.

Además, con el **botón externo** hay dos acciones según cuánto tiempo se mantenga pulsado:
- **De 2 a 3 s**: cancela la operación y vuelve al inicio del modo SERVICIO, sin reiniciar la placa.
- **5 s o más**: entra o sale del modo ADMIN (se activa en cuanto se cumplen los 5 s, sin esperar a soltar).

### Modo ADMIN
El menú tiene 4 opciones, que se recorren con el joystick y se validan con su pulsador. Moviendo el joystick **a la izquierda** se retrocede desde cualquier vista.
- **Ver temperatura**: temperatura y humedad del DHT11 actualizadas en tiempo real.
- **Ver distancia sensor**: distancia en cm, refrescada continuamente.
- **Ver contador**: segundos desde el encendido, avanzando segundo a segundo.
- **Modificar precios**: arriba/abajo cambia el precio en pasos de **0,05 €**, el pulsador **confirma** y la izquierda **descarta**.

Al editar un precio se trabaja con una **copia temporal**: solo al confirmar se escribe en el catálogo. Los cambios se ven después en el modo SERVICIO y se guardan en RAM.

### Recursos utilizados y por qué
**Threads (ArduinoThread)**: Usamos threads para gestionar los intervalos de tiempo de 3 tareas esporadicas a traves del `ThreadController`: 

| Thread | Intervalo | Por qué ese valor |
|---|---|---|
| Distancia | 200 ms | Tiempo mas que suficiente para que pueda calcular la distancia |
| DHT11 | 1500 ms | El DHT11 admite como máximo 1 lectura por segundo, con este tiempo aseguramos un funcionamiento optimo |
| Estado | 50 ms | Para tener respuestas rápidas a las pulsaciones de los botones y realizar acorde los cambios de estado |

**Interrupción hardware**: el botón externo está en el **pin 2** y se captura con `attachInterrupt(..., CHANGE)`. La ISR es muy corta: aplica un antirrebote de 30 ms y guarda el instante de pulsar y de soltar para calcular la duración, que es lo que distingue cancelar (2 a 3 s) de Admin (5 s o más). Las variables compartidas con la ISR son `volatile` y se leen entre `noInterrupts()` e `interrupts()`.

**TimerOne**: `Timer1` genera una interrupción exacta cada segundo e incrementa el contador de "Ver contador".

**Watchdog**: configurado a 4 s con `wdt_enable(WDTO_4S)` y refrescado con `wdt_reset()` en cada vuelta del `loop()`. Si el programa se bloquea, la placa se reinicia sola. Ademas de 4 segundos es mas que tiempo suficiente para todos los procesos que tenemos

**PWM**: el LED2 usa el pin 11 con `analogWrite`. El brillo es proporcional al progreso con `map(elapsed, 0, duración, 0, 255)`.

**Sensores**:
- **HC-SR04**: Donde se calucula la  distancia como distancia = `duración × 0,0343 / 2` (cm). Se divide entre 2 porque el pulso va y vuelve, y le damos un Timeout de 30 ms para no bloquear el programa.
- **DHT11**: si una lectura falla se conserva la última buena; mientras no haya ninguna correcta, el LCD muestra `--`.

### Decisiones de diseño
- **Botón externo en el pin 2**: el ArduinoUNO solo tiene interrupción en los pines 2 y 3, y el 3 lo ocupa el LCD (D7).
- **DHT11 en A3**: se evita el pin 13 porque el LED de la placa carga la línea de datos.
- **Cancelación al soltar**: hasta que no se suelta el botón no se sabe si la pulsación será de 2 s, de 3 s o de más de 5 s.

### Estructura del código
- **Practica3.ino**: `setup()` y `loop()`.
- **Config.hpp**: pines, tiempos, umbrales y catálogo.
- **Sensors.hpp**: lectura del HC-SR04 y del DHT11.
- **Display.hpp**: pantallas del LCD.
- **Input.hpp**: botón externo (interrupción) y joystick.
- **StateMachine.hpp**: modos Arranque, Servicio y Admin, y contador con Timer1.
- **Threads.hpp**: tareas periódicas con ArduinoThread.

Librerías necesarias: LiquidCrystal, ArduinoThread, DHT sensor library (Adafruit) con Adafruit Unified Sensor, y TimerOne.

