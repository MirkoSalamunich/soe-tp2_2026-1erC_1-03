# TP2 – Actividad 01 – 4to Proyecto p/placa NUCLEO-F103RB con FreeRTOS

## Paso 06: 

## 1. Respuesta de la IA Gemini con el prompt:

## Analizar y explicar (en español), el funcionamiento del código fuente contenido en los archivos adjuntos: startup_stm32f103rbtx.s , main.c , stm32f1xx_it.c , FreeRTOSConfig.h y freertos.c . Indicar la evolución de las variables SysTick y SystemCoreClock al ejecutar dicho código fuente desde su inicio ( Reset_Handler : de startup_stm32f103rbtx.s ) hasta el loop principal de la aplicación ( while (1) de main.c ). Indicar el comportamiento del programa al ejecutar dicho código fuente desde su inicio ( Reset_Handler : de startup_stm32f103rbtx.s ) hasta antes de llegar al loop principal de la aplicación ( while (1) de main.c ). Indicar cómo y para qué SysTick y Timer 1 ( TIM2 ) interactúan con FreeRTOS . Indicar cómo y para qué el Timer 4 ( TIM4 ) interactúa con la HAL del proyecto STM32 .

Aquí tienes el análisis detallado del funcionamiento del código y las respuestas a tus consultas sobre la arquitectura del proyecto:

### Análisis General de los Archivos

* **startup_stm32f103rbtx.s**: Es el archivo de arranque (escrito en lenguaje ensamblador) que toma el control del microcontrolador inmediatamente después de un reinicio. Define la tabla de vectores de interrupción, inicializa el puntero de pila (Stack Pointer), copia los valores iniciales de las variables a la memoria RAM (sección `.data`), inicializa en cero la memoria no predefinida (sección `.bss`), llama a la configuración inicial del sistema y finalmente hace el salto a la función `main()`.
* **main.c**: Contiene el punto de entrada principal de la aplicación. Aquí se inicializa la capa de abstracción de hardware (HAL), se configura el árbol de relojes del sistema y se inicializan los periféricos específicos (GPIO, UART2, y TIM2). Además, se encarga de crear las tareas iniciales del sistema operativo (`defaultTask`) e iniciar el planificador de FreeRTOS llamando a `osKernelStart()`.
* **stm32f1xx_it.c**: Contiene las rutinas de servicio de interrupción (ISR). Sirve como puente entre los eventos de hardware y la lógica de la aplicación. Por ejemplo, delega las interrupciones de los temporizadores TIM1 y TIM2 a los manejadores propios de la HAL (`HAL_TIM_IRQHandler`), y gestiona las interrupciones externas (EXTI) de los botones físicos.
* **FreeRTOSConfig.h**: Es el archivo central de configuración de FreeRTOS. Define parámetros vitales como la frecuencia del Tick del sistema (1000 Hz), el tamaño del Heap, soporte para reserva de memoria estática y dinámica, y habilita herramientas avanzadas como las estadísticas de tiempo de ejecución (`configGENERATE_RUN_TIME_STATS`). También mapea interrupciones vitales del núcleo ARM Cortex-M (como SysTick y PendSV) a las funciones internas del RTOS.
* **freertos.c**: Actúa como un espacio para las extensiones y ganchos (hooks) del RTOS. En este código, provee la memoria estática necesaria para la tarea inactiva (Idle Task) del RTOS mediante `vApplicationGetIdleTaskMemory` y expone las funciones necesarias para llevar la cuenta de las estadísticas de ejecución utilizando los contadores de hardware.

---

### Evolución de `SysTick` y `SystemCoreClock`

* **`SystemCoreClock`**: Al arrancar desde el `Reset_Handler`, la variable no tiene su valor definitivo. Toma un valor base inicial cuando se llama a `SystemInit()`. Más tarde, en `main.c`, la función `SystemClock_Config()` configura los osciladores internos (HSI) y el bucle de enganche de fase (PLL) para multiplicar la frecuencia. Una vez finalizada esta función, `SystemCoreClock` refleja la velocidad de reloj máxima configurada para el procesador.
* **`SysTick`**: Es el temporizador del núcleo Cortex-M. No se activa en el arranque inmediato. Aunque la inicialización típica de la HAL (`HAL_Init()`) suele iniciarlo, en un entorno FreeRTOS, el SysTick está mapeado para uso exclusivo del RTOS (`#define xPortSysTickHandler SysTick_Handler`). Permanece inactivo o bajo el control de la HAL de forma temporal hasta que se invoca `osKernelStart()`. En ese preciso instante, FreeRTOS toma control del `SysTick` para generar interrupciones regulares (a 1000 Hz) que dictan los cambios de contexto.

---

### Comportamiento del programa (Desde `Reset_Handler` hasta el `while(1)`)

1. El microcontrolador recibe energía y el hardware ubica el vector de inicio para saltar a `Reset_Handler`.
2. Se copian los datos iniciados desde la memoria Flash a la SRAM y se limpian las variables no iniciadas.
3. Se ejecuta un salto a la función `main()`.
4. Se invoca `HAL_Init()` y luego `SystemClock_Config()` para establecer la velocidad y periféricos base.
5. Se configuran los puertos de E/S, la comunicación serial (UART) y el temporizador 2 (TIM2), el cual arranca explícitamente mediante `HAL_TIM_Base_Start_IT(&htim2)`.
6. Se define y crea un hilo base del sistema (`defaultTaskHandle`) en la RAM.
7. Se llama a `osKernelStart()`, activando el SysTick del RTOS y entregando el control del microcontrolador al planificador de tareas.
8. **Punto Crítico:** El bucle `while (1)` ubicado al final de `main()` **nunca se ejecuta**. Esto es intencional; debido a que `osKernelStart()` secuestra el hilo de ejecución para correr las tareas de FreeRTOS, el flujo normal de `main()` solo continuaría (llegando al `while(1)`) si el RTOS falla catastróficamente al iniciar (por ejemplo, si se queda sin memoria RAM para la primera tarea).

---

### Interacción de `SysTick`, `Timer 1 (TIM1)` y `Timer 2 (TIM2)` con FreeRTOS

Para mayor claridad, separaré los periféricos, ya que cumplen funciones totalmente distintas en tu arquitectura:

* **SysTick:** Está configurado para ser el "latido" (Tick) de FreeRTOS. Su interrupción ejecuta código interno del kernel operativo que evalúa qué tareas deben despertar (como tu `defaultTask` al usar `osDelay(1)`) o si existe una tarea de mayor prioridad esperando a ser procesada.
* **Timer 2 (TIM2):** Funciona como un temporizador de alta frecuencia dedicado *exclusivamente* a las estadísticas de uso de CPU de FreeRTOS. Su interrupción incrementa la variable `ulHighFrequencyTimerTicks`. FreeRTOS utiliza la función `getRunTimeCounterValue()` para leer este contador y calcular qué porcentaje de tiempo de CPU gasta cada tarea individual.
* **Timer 1 (TIM1):** Debido a que FreeRTOS "roba" el SysTick estándar, las librerías base de STM32 (la HAL) necesitan su propia referencia de tiempo aislada para manejar los "timeouts" o tiempos de espera de los periféricos hardware. El código redirige el contador base de la HAL (`HAL_IncTick()`) para que se actualice cada vez que se dispara el TIM1. TIM1 interactúa con la HAL, no con FreeRTOS.

---

### Interacción del Timer 4 (TIM4) con la HAL del proyecto

Basado en el código fuente proporcionado, **el temporizador 4 (TIM4) no interactúa en absoluto ni con la HAL, ni con el RTOS, ni es utilizado por el sistema**.

Solo existe una referencia a `TIM4_IRQHandler` en la tabla de vectores del archivo `startup_stm32f103rbtx.s` declarada como una rutina inactiva (weak alias apuntando a `Default_Handler`) por defecto de fábrica. En los archivos de inicialización y control (`main.c` y `stm32f1xx_it.c`) no hay rastros de configuración (`MX_TIM4_Init`), rutinas ISR activadas, o llamadas a funciones HAL para habilitar este temporizador en particular.



# -------------------------- # ------------------------- # -------------------------- # ------------------------- # -------------------------- # ------------------------- #
## 2. Respuesta de la IA Gemini con el prompt:

## Analizar y explicar (en español), el funcionamiento del código fuente contenido en los archivos adjuntos: app.c , task_btn.c , task_led.c , task_led_interface.c , y freertos.c .

Este conjunto de archivos implementa un sistema embebido basado en eventos (Event-Triggered System) utilizando el sistema operativo en tiempo real FreeRTOS. El objetivo principal del código es leer el estado de un botón (con supresión de rebotes o *debouncing*) y controlar el parpadeo de un LED mediante máquinas de estados finitos que se ejecutan en tareas separadas.

A continuación, se detalla el funcionamiento de cada archivo:

### 1. `freertos.c` (Ganchos del Sistema / System Hooks)

Este archivo contiene las funciones de "gancho" (hooks) que FreeRTOS llama automáticamente bajo ciertas condiciones del sistema operativo.

* La función `vApplicationIdleHook()` se ejecuta cuando no hay tareas de mayor prioridad listas para correr (tarea Idle) e incrementa el contador `g_task_idle_cnt`.
* La función `vApplicationTickHook()` se ejecuta en cada interrupción del "tick" (el reloj del sistema operativo) e incrementa el contador `g_app_tick_cnt`.
* La función `vApplicationStackOverflowHook()` se activa si el sistema detecta que una tarea ha desbordado su pila (stack), deteniendo la ejecución con `configASSERT( 0 )` para facilitar la depuración e incrementando el contador `g_app_stack_overflow_cnt`.

### 2. `app.c` (Inicialización de la Aplicación)

Este archivo actúa como el punto de entrada para la configuración de las tareas del usuario.

* La función `app_init()` inicializa los contadores de la aplicación en cero.
* Se utiliza la función `xTaskCreate` para instanciar dos tareas concurrentes: `Task BTN` (para el botón) y `Task LED` (para el LED).
* Ambas tareas son creadas con el mismo nivel de prioridad (`tskIDLE_PRIORITY + 1ul`).
* La función verifica mediante `configASSERT` que ambas tareas se hayan creado exitosamente en la memoria antes de continuar.

### 3. `task_btn.c` (Lectura del Botón y Máquina de Estados)

Este archivo contiene la lógica encargada de monitorear un botón físico y filtrar los ruidos mecánicos.

* La tarea principal `task_btn()` se ejecuta en un bucle infinito con un retraso de 50 milisegundos (`BTN_TICK_DEL_MAX`) utilizando `vTaskDelay`.
* La lógica interna es controlada por la función `task_btn_statechart()`, la cual evalúa el estado físico del pin mediante `HAL_GPIO_ReadPin`.
* La máquina de estados transita por cuatro etapas para evitar falsas lecturas: `ST_BTN_UP`, `ST_BTN_FALLING`, `ST_BTN_DOWN` y `ST_BTN_RISING`.
* Si el sistema confirma que el botón ha sido presionado sólidamente (pasando de `ST_BTN_FALLING` a `ST_BTN_DOWN`), envía la orden de parpadeo ejecutando `put_event_task_led(EV_LED_BLINK)`.
* Si el sistema confirma que el botón ha sido soltado (pasando de `ST_BTN_RISING` a `ST_BTN_UP`), envía la orden de apagado ejecutando `put_event_task_led(EV_LED_OFF)`.

### 4. `task_led_interface.c` (Comunicación Inter-Tareas)

Este archivo proporciona una interfaz simple para enviar eventos desde la tarea del botón hacia la tarea del LED.

* La función `put_event_task_led()` recibe un evento y actualiza la estructura de datos del LED asignando dicho evento a la variable `task_led_dta.event`.
* La misma función levanta una bandera lógica (asignando `true` a `task_led_dta.flag`) para notificar a la máquina de estados del LED que hay un nuevo comando pendiente por procesar.

### 5. `task_led.c` (Control del LED y Máquina de Estados)

Este archivo es responsable de encender, apagar o hacer parpadear el LED en respuesta a los eventos recibidos.

* La tarea principal `task_led()` se ejecuta en un bucle infinito y garantiza una periodicidad estricta de 50 milisegundos utilizando `vTaskDelayUntil`.
* La función `task_led_statechart()` maneja la lógica de control del LED basándose en dos estados principales: `ST_LED_OFF` y `ST_LED_BLINK`.
* En el estado `ST_LED_OFF`, si la bandera es verdadera y el evento es `EV_LED_BLINK`, la tarea enciende el LED físicamente, reinicia su contador de tiempo y cambia al estado `ST_LED_BLINK`.
* En el estado `ST_LED_BLINK`, la tarea alterna (hace parpadear) el estado del pin cada 500 milisegundos (`DEL_LED_MAX`) usando `HAL_GPIO_TogglePin`.
* Si estando en `ST_LED_BLINK` se recibe un evento `EV_LED_OFF`, la máquina apaga el LED inmediatamente y regresa al estado de reposo `ST_LED_OFF`.