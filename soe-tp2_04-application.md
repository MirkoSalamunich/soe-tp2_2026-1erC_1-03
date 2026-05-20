# TP2 – Actividad 04 – 7mo Proyecto p/placa NUCLEO-F103RB con FreeRTOS

## Paso 02: 

## ¿Qué funciones de la API de FreeRTOS se pueden usar dentro de una rutina de servicio de interrupción?
Solo aquellas que terminan con el sufijo `FromISR`. No se deben usar funciones bloqueantes (como las que no tienen el sufijo) porque las ISR no pueden bloquearse.

Estas funciones están diseñadas para ser determinísticas y no ceder el control de la CPU directamente. Un parámetro clave que utilizan es `pxHigherPriorityTaskWoken`, que indica si la interrupción despertó a una tarea de mayor prioridad que la actual.
`xQueueSendFromISR()`, `xSemaphoreGiveFromISR()`, `vTaskNotifyGiveFromISR()`.

## ¿Métodos para delegar el procesamiento de interrupciones a una Tarea?
Se utiliza el patrón de "Deferred Interrupt Processing" (Procesamiento de interrupción diferido). La ISR hace el trabajo mínimo (limpiar flags) y notifica a una tarea para que haga el trabajo pesado.

Los mecanismos más comunes son los Semáforos Binarios (como hicimos en este TP) o las Notificaciones de Tarea (más rápidas). La tarea se queda bloqueada en un `xSemaphoreTake()` o `ulTaskNotifyTake()` hasta que la ISR le da el "visto bueno", liberando la CPU para otras tareas mientras no hay eventos.


## ¿Cómo usar una cola para transferir datos dentro y fuera de una rutina de servicio de interrupción?
Se utiliza `xQueueSendFromISR()` para enviar datos desde la ISR a una tarea, y `xQueueReceiveFromISR()` para lo inverso (aunque lo más común es de ISR a Tarea).

La ISR no puede esperar si la cola está llena, por lo que el tiempo de bloqueo siempre debe ser 0. Es un método seguro para pasar información estructurada (como valores de un ADC o caracteres de una UART) sin riesgo de condiciones de carrera.

BaseType_t xHigherPriorityTaskWoken = pdFALSE;
xQueueSendFromISR(h_Queue, &dato, &xHigherPriorityTaskWoken);
portYIELD_FROM_ISR(xHigherPriorityTaskWoken);

## ¿Cuál es el modelo de anidamiento de interrupciones disponible en algunas portaciones de FreeRTOS?
Es el modelo de "Prioridades Segmentadas", definido por las macros `configMAX_SYSCALL_INTERRUPT_PRIORITY` y `configKERNEL_INTERRUPT_PRIORITY`.

Las interrupciones con prioridad numérica menor (mayor urgencia) que el "Syscall Priority" no pueden usar funciones de la API de FreeRTOS, pero tienen latencia cero. Las que tienen prioridad mayor (menor urgencia) pueden usar la API y pueden ser anidadas por las de mayor prioridad. Esto evita que el kernel corrompa sus datos internos mientras se ejecutan tareas críticas de hardware.


# -------------------------- # ------------------------- # -------------------------- # ------------------------- # -------------------------- # ------------------------- #
# Paso 03:

### Modificación de task_btn para gestión del botón por interrupción y semáforo binario

- **Creación del Semáforo:** En `app.c` se declaró y creó un semáforo binario `h_sem_btn_event` usando `xSemaphoreCreateBinary()`.

- **Sincronización en la Tarea:** Se modificó el bucle principal de `task_btn()` en `task_btn.c`. Se reemplazó el `vTaskDelay()` por `xSemaphoreTake()`.

- **Bloqueo Dinámico:** Se implementó una lógica de espera condicional: si el botón está en reposo (`ST_BTN_XX_UP`), la tarea se bloquea indefinidamente con `portMAX_DELAY`. Si está en otro estado (procesando rebote), usa el tiempo de tick original.

- **Rutina de Interrupción (ISR):** Se agregó la función `HAL_GPIO_EXTI_Callback()`. Esta función libera el semáforo mediante `xSemaphoreGiveFromISR()` y solicita un cambio de contexto inmediato con `portYIELD_FROM_ISR()`.


**Configuración del .ioc**
- **Pin:** **PC13** (pin default del Botón azul de la placa NUCLEO).
- **Modo del Pin:** Configuración default: **GPIO_EXTI13** (External Interrupt).
- **Tratamiento de flanco:** Configuración: **Falling Edge detection** (detección de flanco descendente). 
La configuración de flaco fue invertida ya que por defecto estaba como **Rising Edge detection** (detección de flanco ascendente), pero de esta forma el sistema no funcionaba.


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
1. **Eficiencia de CPU:** A diferencia del polling, la tarea `task_btn` ahora consume 0% de CPU mientras el botón no es presionado, ya que permanece en estado *Blocked* (esperando el semáforo) sin necesidad de despertar periódicamente.
2. **Sincronización ISR-Tarea:** El semáforo binario actúa como el mecanismo de comunicación ideal para "despertar" una tarea desde una interrupción de hardware de manera segura.
3. **Mantenimiento de la Lógica:** Se logró integrar la interrupción sin modificar la máquina de estados original (`task_btn_statechart`), manteniendo el sistema de anti-rebote (debouncing) por software.
4. **Determinismo:** El uso de `portYIELD_FROM_ISR()` asegura que, ni bien se suelta el botón y se da el semáforo, la tarea de mayor prioridad tome el control de la CPU inmediatamente, reduciendo la latencia de respuesta.


*Extra:* 
Para poder visualizar el semaforo creado mediante la pestaña de FreeRTOS dentro del debbuger, se agregó la linea:
`vQueueAddToRegistry(h_sem_btn_event, "Sem_Btn_Event");`






