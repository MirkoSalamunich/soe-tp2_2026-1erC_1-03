# TP2 – Actividad 03 – 6to Proyecto p/placa NUCLEO-F103RB con FreeRTOS

## Paso 02: 

## ¿Cómo crear y usar semáforos binarios y semáforos contadores?
*   **Creación de Semáforo Binario:** `vSemaphoreCreateBinary(xSemaphore)`.
*   **Creación de Semáforo Contador:** se utiliza la función `xSemaphoreCreateCounting(uxMaxCount, uxInitialCount)`, donde se especifica el valor máximo y el valor inicial del contador.
*   **Tomar el semáforo (Wait):** Se utiliza `xSemaphoreTake(xSemaphore, xTicksToWait)`; esta función bloquea la tarea si el semáforo no está disponible hasta que transcurra el tiempo de espera definido.
*   **Dar el semáforo (Release):** Se utiliza `xSemaphoreGive(xSemaphore)` para liberar el semáforo y permitir que otra tarea lo tome.
*   **Uso en interrupciones:** Para liberar o tomar semáforos dentro de una rutina de interrupción, se deben usar las versiones específicas `xSemaphoreGiveFromISR()` y `xSemaphoreTakeFromISR()`.
*   **Eliminación:** Para borrar un semáforo y liberar su memoria se usa `vSemaphoreDelete(xSemaphore)`.

## ¿Cuáles son las diferencias entre semáforos binarios y semáforos contadores?
Aunque ambos se basan internamente en el mecanismo de colas de FreeRTOS, presentan las siguientes diferencias fundamentales:
*   **Rango de valores:** El semáforo binario funciona como un interruptor de encendido/apagado, teniendo únicamente dos estados posibles: 0 o 1. El semáforo contador puede almacenar múltiples "fichas" o tokens, permitiendo un seguimiento de valores mayores a uno.
*   **Propósito de uso:** Los semáforos binarios son la mejor opción para implementar **sincronización** simple entre tareas o entre una tarea y una interrupción. Los semáforos contadores se utilizan típicamente para **gestión de recursos** (donde el valor indica cuántas unidades de un recurso están disponibles) o para el **conteo de eventos** (donde el valor indica la diferencia entre eventos ocurridos y procesados).
*   **Configuración de longitud:** Un semáforo binario es, de hecho, una cola con una longitud de 1 y un tamaño de datos de 0. Un semáforo contador se comporta como una cola con una longitud mayor a 1, donde al usuario no le interesan los datos enviados, sino si la cola está vacía o no.
*   **Valor inicial:** En el conteo de eventos, el semáforo contador suele crearse con un valor inicial de 0. En la gestión de recursos, es deseable que el valor inicial sea igual al valor máximo permitido al momento de su creación.

# -------------------------- # ------------------------- # -------------------------- # ------------------------- # -------------------------- # ------------------------- #
# Paso 03:

### Modificaciones para la gestión de la comunicación entre tareas mediante semáforo binario
- **Creación del Semáforo:** En `app.c` se declaró el manejador `h_sem_led_event` de tipo `SemaphoreHandle_t` de forma global y se inicializó dentro de `app_init()` utilizando la función nativa `xSemaphoreCreateBinary()`.
- **Sincronización desde la Interfaz (Give):** En `task_led_interface.c`, dentro de la función `put_event_task_led()`, se eliminó la asignación directa sobre la variable compartida `task_led_dta.flag = true`. En su lugar, se insertó la llamada a `xSemaphoreGive(h_sem_led_event)` para notificar de manera segura a la tarea receptora. Se preservó la asignación del tipo de evento en `task_led_dta.event`.
- **Recepción No Bloqueante (Take):** En `task_led.c`, al inicio de la función `task_led_statechart()`, se implementó la lógica de lectura mediante `xSemaphoreTake(h_sem_led_event, 0)`. Al utilizar un tiempo de espera de 0 (no bloqueante), la tarea comprueba instantáneamente si el botón envió una señal. Si el semáforo fue tomado (`pdTRUE`), se activa internamente la bandera local para mantener la compatibilidad con el switch-case subsiguiente.


### Resultado en la terminal
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
1. **Sincronización sobre Variables Compartidas:** Reemplazar el flag manual por un semáforo binario elimina los riesgos asociados al acceso concurrente desprotegido entre `task_btn` y `task_led`, delegando la exclusión mutua y la señalización directamente al micronúcleo de FreeRTOS.
2. **Importancia del Enfoque No Bloqueante:** La tarea `task_led` tiene requerimientos temporales estrictos y cíclicos controlados por `vTaskDelayUntil()`. Si se hubiera configurado el semáforo en modo bloqueante (`portMAX_DELAY`), la ejecución de la tarea se detendría por completo al apagar el LED, impidiendo que efectúe el parpadeo periódico autónomo de 50ms.
3. **Limitación de Datos en Semáforos:** Dado que el semáforo binario es solo una bandera lógica abstracta y no posee un buffer de almacenamiento, sigue siendo necesario mantener la variable estructural `task_led_dta.event` para que la tarea LED distinga qué acción específica (`BLINK` u `OFF`) le está solicitando el botón.
4. **Preservación de la Máquina de Estados:** Mediante el mapeo interno (`task_led_dta.flag = true`) al momento de tomar con éxito el semáforo, se evitó modificar la compleja estructura del diagrama de estados original, logrando el desacoplamiento requerido con la menor cantidad de líneas de código alteradas.





