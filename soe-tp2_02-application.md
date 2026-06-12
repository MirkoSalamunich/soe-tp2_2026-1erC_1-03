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

---
---

## Paso 03:

### Modificaciones para la gestión de la comunicación entre tareas mediante una Cola (Queue)
- **Creación de la Cola:** En `app.c` se declaró el handle `h_button_queue` de tipo `QueueHandle_t` de forma global y se inicializó dentro de `app_init()` utilizando la función nativa `xQueueCreate(5, sizeof(uint8_t))`. Esta configuración asigna un buffer con capacidad para almacenar hasta 5 elementos de un byte cada uno de manera segura.
- **Transmisión desde la Tarea Emisora (Send):** En `task_btn.c`, dentro de la máquina de estados y específicamente al confirmarse el flanco en el estado `ST_BTN_XX_FALLING`, se eliminó por completo la dependencia con la función externa `put_event_task_led()`. En su lugar, se insertó la llamada a `xQueueSend(h_button_queue, &send_event, 0)` para inyectar un código de evento de forma directa y no bloqueante en el almacenamiento intermedio de la cola.
- **Recepción No Bloqueante en la Tarea Receptora (Receive):** En `task_led.c`, al inicio de la función `task_led_statechart()`, se implementó la lógica de lectura mediante `xQueueReceive(h_button_queue, &ev_recibido, 0)`. Al utilizar un tiempo de espera de 0 (no bloqueante), la tarea comprueba instantáneamente si ingresó un nuevo mensaje. Si se extrae un elemento con éxito (`pdTRUE`), se inyecta el evento `EV_LED_XX_BLINK` y se activa localmente `task_led_dta.flag = true` para mantener la compatibilidad con el switch-case subsiguiente.

*Resultado terminal:*
```
[info]
[info] app_init is running - Tick [mS] =   0
[info]  RTOS - Event-Triggered Systems (ETS)
[info]  soe-tp2_02-application: Demo Code
[info]
[info] Task LED is running - Tick [mS] =   0
[info]
[info] Task BTN is running - Tick [mS] =   0
[info]  Task BTN - BTN PRESSED
[info]  Task LED - LED BLINK
[info]  Task BTN - BTN HOVER
[info]  Task LED - LED OFF
```

**Observaciones:**
1. **Desacoplamiento de Tareas mediante IPC:** El uso de una cola de FreeRTOS reemplaza de forma nativa e integral el archivo de acoplamiento indirecto `task_led_interface.c`. Esto permite que la tarea del botón envíe información de manera asincrónica sin necesidad de modificar variables desprotegidas en el entorno de la tarea del LED.
2. **Importancia del Enfoque No Bloqueante (Timeout = 0):** La tarea `task_led` posee requerimientos temporales estrictos y cíclicos controlados por `vTaskDelayUntil()` en su bucle principal. Si se hubiera configurado el `xQueueReceive` en modo bloqueante (`portMAX_DELAY`), la ejecución se congelaría indefinidamente esperando al botón, impidiendo el procesamiento de los desbordamientos por tiempo (`DEL_LED_XX_MAX`) necesarios para ejecutar el parpadeo periódico.
3. **Paso de Datos e Identificación de Eventos:** A diferencia de un semáforo (que actúa como un simple token abstracto), la cola transporta valores reales (`sizeof(uint8_t)`). Aunque en este paso inicial se utiliza un valor genérico para activar la bandera, este mecanismo queda preparado formalmente para escalar hacia el envío de tramas complejas o múltiples tipos de comandos embebidos dentro del mismo canal de comunicación.
4. **Preservación de la Máquina de Estados:** Mediante el mapeo interno de variables estructurales (`flag = true` y `event = EV_LED_XX_BLINK`) al momento de retirar un dato válido de la cola, se garantizó la integridad del diagrama de estados original diseñado por la cátedra, logrando una reingeniería limpia y con un impacto mínimo en el código existente.
