# TP2 – Actividad 02 – 5to Proyecto p/placa NUCLEO-F103RB con FreeRTOS

## Paso 02: 

## ¿Cómo eliminar una Cola?
Para eliminar una cola y liberar la memoria asociada se utiliza la función nativa `vQueueDelete(xQueue)`.

## ¿Cómo crear una Cola?
Se utiliza la función `xQueueCreate(uxQueueLength, uxItemSize)`. Esta función solicita memoria al Heap de FreeRTOS para crear la estructura de la cola y su buffer de datos. También existe `xQueueCreateStatic()` para aplicaciones que no utilizan asignación de memoria dinámica.

## ¿Cómo gestiona una Cola los datos que contiene?
*   **Por Copia:** Los datos se copian físicamente dentro de la cola (en lugar de solo guardar un puntero), lo que permite que la tarea emisora sobrescriba su variable original inmediatamente después de enviarla.
*   **Orden FIFO:** Por defecto, el primer dato en entrar es el primero en salir (FIFO).
*   **Capacidad de LIFO:** Puede configurarse para funcionar como una pila (LIFO) enviando datos al frente de la cola.
*   **Tipos Uniformes:** Todos los elementos deben tener el mismo tamaño, definido al crear la cola.

## ¿Cómo enviar datos a una Cola?
Se utilizan las siguientes funciones:
*   `xQueueSend()` o `xQueueSendToBack()`: Envía el dato al final de la cola (orden FIFO).
*   `xQueueSendToFront()`: Envía el dato al principio de la cola (orden LIFO).
*   `xQueueSendFromISR()`: Versión específica para ser utilizada dentro de rutinas de interrupción.

## ¿Cómo recibir datos de una Cola?
*   **Extraer:** Se usa `xQueueReceive()`, que lee el dato y lo elimina de la cola. En interrupciones se usa `xQueueReceiveFromISR()`.
*   **Observar (Peek):** Se usa `xQueuePeek()`, que permite leer el dato sin quitarlo de la cola, dejándolo disponible para la siguiente lectura.

## ¿Qué significa bloquearse en una Cola?
Significa que una tarea entra en estado **Blocked** y no consume tiempo de CPU mientras espera que cambie el estado de la cola. 
*   Una tarea se bloquea al intentar leer si la cola está vacía.
*   Una tarea se bloquea al intentar escribir si la cola está llena.
El tiempo máximo de espera se define mediante el parámetro `xTicksToWait`.

## ¿Cómo bloquearse en varias Colas?
FreeRTOS ofrece el mecanismo de **Queue Sets** (conjuntos de colas) para que una tarea pueda bloquearse esperando que llegue un dato a cualquiera de las colas que integran el conjunto. Para esto, se debe habilitar la configuración `configUSE_QUEUE_SETS` en el archivo `FreeRTOSConfig.h`.

## ¿Cómo sobrescribir datos en una Cola?
Aunque las fuentes mencionan principalmente el envío al frente o atrás, en la API nativa de FreeRTOS se utiliza habitualmente la función `xQueueOverwrite()` (especialmente útil en colas de longitud 1) para reemplazar el dato existente por uno nuevo, incluso si la cola ya está llena.

## ¿Cómo vaciar una Cola?
Se puede vaciar una cola mediante lecturas sucesivas con `xQueueReceive()` hasta que la cola esté vacía (lo cual se puede verificar con `uxQueueMessagesWaiting()`).

## ¿Cuál es el efecto de las prioridades de las Tareas al escribir y leer en una Cola?
Las prioridades determinan el orden de acceso cuando varias tareas están bloqueadas esperando la misma cola:
*   Si varias tareas esperan para leer, cuando llega un dato, el Scheduler desbloquea a la **tarea de mayor prioridad** que esté en espera.
*   Si las tareas tienen igual prioridad, se desbloquea a la que **más tiempo lleve esperando**. 

# -------------------------- # ------------------------- # -------------------------- # ------------------------- # -------------------------- # ------------------------- #
# Paso 03:  Sincronización y comunicación concurrente entre tareas de control

### Configuracion
**Modificaciones y lógica de diseño en el código:**
- **Resolución del bloqueo por inanición (Starvation):** Se corrigió el comportamiento del paso previo donde una tarea monopolizaba el uso de la CPU. Para ello, se estructuraron ambas tareas corporativas bajo un modelo de multitarea apropiativa basada en tiempo, garantizando que cedan el control del procesador de forma voluntaria cuando no tienen trabajo útil que realizar.
- **Implementación de periodicidad estricta en la Tarea LED:** En `task_led.c`, dentro del bucle infinito de la tarea, se implementó la función nativa `vTaskDelayUntil()`. Esto asegura que la tarea mantenga un período de ejecución constante (fijado por `LED_TICK_DEL_MAX` en 50ms) y, lo más importante, que pase automáticamente al estado **Blocked** (Bloqueado) durante el intervalo de espera, liberando el 100% de los recursos de la CPU para otras tareas.
- **Estructuración de la Tarea de Botón (Debouncing No Bloqueante):** En `task_btn.c`, la lógica de lectura y filtrado de rebotes por software se delegó a una máquina de estados (`task_btn_statechart`). En lugar de utilizar retardos bloqueantes tradicionales dentro del switch-case (lo cual congelaría el sistema), la tarea utiliza `xTaskGetTickCount()` para medir el tiempo transcurrido de forma asincrónica, complementado con un retardo periódico en su bucle principal para permitir la alternancia del contexto.
- **Mecanismo de comunicación por interfaz (Desacoplamiento):** La comunicación se estructuró a través del archivo de interfaz `task_led_interface.c`. Cuando la máquina de estados del botón detecta una transición válida (por ejemplo, de `ST_BTN_XX_FALLING` a `ST_BTN_XX_DOWN`), invoca a la función `put_event_task_led()`. Esta función encapsula la escritura segura sobre la estructura global compartida, asignando el tipo de evento (`EV_LED_XX_BLINK` o `EV_LED_XX_OFF`) y activando la bandera de notificación lógica (`task_led_dta.flag = true`).

*Resultado*
[info]

[info] app_init is running - Tick [mS] =   0
[info]  RTOS - Event-Triggered Systems (ETS)
[info]  soe-tp0_03-application: Demo Code
[info]

[info] Task LED is running - Tick [mS] =   0
[info]

[info] Task BTN is running - Tick [mS] =   0
[info]  Task BTN - BTN PRESSED
[info]  Task LED - LED BLINK
[info]  Task BTN - BTN HOVER
[info]  Task LED - LED OFF

**Observaciones:**
1. **Inicialización Concurrente Exitosa:** Al arrancar el sistema (`Tick [mS] = 0`), el planificador (*scheduler*) ejecuta de forma alternada los bloques de inicialización de ambas tareas. Se constata la correcta creación y puesta en marcha de `Task LED` y `Task BTN` mediante el vaciado de sus respectivos mensajes de diagnóstico en el canal de log.
2. **Conmutación de Contexto Efectiva:** Se verifica experimentalmente la resolución del bloqueo por bucle infinito. Al compartir la misma prioridad (`tskIDLE_PRIORITY + 1ul`), el scheduler realiza un reparto de tiempo (*Time Slicing*) eficiente. Cuando una tarea entra en estado de bloqueo temporal mediante las funciones de delay del RTOS, el contexto conmuta inmediatamente hacia la otra tarea lista para ejecutar.
3. **Flujo y Secuencia de Eventos:** La interacción entre el estímulo físico y la respuesta lumínica responde fielmente a las transiciones del diagrama de estados:
   - Al presionar el pulsador, la tarea de botón procesa el flanco descendente, filtra el rebote y loguea `BTN PRESSED`, disparando el evento de parpadeo a través de la interfaz.
   - La tarea de LED, al despertar en su respectivo ciclo de 50ms, evalúa la bandera, consume el evento y loguea `LED BLINK`, modificando el estado del pin físico de la placa.
   - Al liberarse el pulsador, el flujo asocia el estado de reposo (`BTN HOVER`) con la desactivación del periférico (`LED OFF`), demostrando una sincronización limpia y determinística.