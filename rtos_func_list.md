# FreeRTOS API Function Reference (with Signatures)

Organized by category to match the JD responsibility areas. This is the exact syntax — know at least the parameter meanings for the ones marked ⭐ (most likely to come up given your project and the JD).

---

## 1. Task Management

**⭐ xTaskCreate** — create a task, dynamic memory allocation
```c
BaseType_t xTaskCreate(
    TaskFunction_t pvTaskCode,     // pointer to task entry function
    const char * const pcName,     // descriptive name (debug only)
    uint16_t usStackDepth,         // stack size in WORDS (not bytes, in vanilla FreeRTOS)
    void *pvParameters,            // pointer passed into the task function
    UBaseType_t uxPriority,        // task priority (0 = lowest)
    TaskHandle_t *pxCreatedTask    // out: handle to created task
);
// Returns: pdPASS on success, error code otherwise
```

**xTaskCreateStatic** — create a task without dynamic allocation (you supply the memory)
```c
TaskHandle_t xTaskCreateStatic(
    TaskFunction_t pvTaskCode,
    const char * const pcName,
    uint32_t ulStackDepth,
    void *pvParameters,
    UBaseType_t uxPriority,
    StackType_t *puxStackBuffer,   // caller-provided stack memory
    StaticTask_t *pxTaskBuffer     // caller-provided TCB memory
);
```

**⭐ vTaskDelete** — delete a task
```c
void vTaskDelete( TaskHandle_t xTaskToDelete );  // NULL = delete calling task
```

**⭐ vTaskDelay** — block for a fixed number of ticks (relative delay)
```c
void vTaskDelay( const TickType_t xTicksToDelay );
```

**vTaskDelayUntil** — block until an absolute time (better for periodic tasks — avoids drift)
```c
void vTaskDelayUntil(
    TickType_t *pxPreviousWakeTime,  // in/out: last wake time
    const TickType_t xTimeIncrement  // period
);
```

**⭐ vTaskSuspend / vTaskResume** — pause/resume a task
```c
void vTaskSuspend( TaskHandle_t xTaskToSuspend );
void vTaskResume( TaskHandle_t xTaskToResume );
BaseType_t xTaskResumeFromISR( TaskHandle_t xTaskToResume );  // ISR-safe variant
```

**⭐ vTaskPrioritySet / uxTaskPriorityGet** — change/read task priority at runtime
```c
void vTaskPrioritySet( TaskHandle_t xTask, UBaseType_t uxNewPriority );
UBaseType_t uxTaskPriorityGet( const TaskHandle_t xTask );
```

**⭐ uxTaskGetStackHighWaterMark** — check peak stack usage (validation-relevant!)
```c
UBaseType_t uxTaskGetStackHighWaterMark( TaskHandle_t xTask );
// Returns: minimum free stack space ever recorded, in words — low value = near overflow
```

**vTaskGetInfo** — detailed task diagnostics (state, priority, stack usage, etc.)
```c
void vTaskGetInfo(
    TaskHandle_t xTask,
    TaskStatus_t *pxTaskStatus,
    BaseType_t xGetFreeStackSpace,
    eTaskState eState
);
```

**taskYIELD** — force an immediate reschedule (macro, not function)
```c
taskYIELD();
```

---

## 2. Queues (IPC)

**⭐ xQueueCreate** — create a queue
```c
QueueHandle_t xQueueCreate(
    UBaseType_t uxQueueLength,   // max number of items
    UBaseType_t uxItemSize       // size of each item in bytes
);
```

**⭐ xQueueSend / xQueueSendToBack / xQueueSendToFront**
```c
BaseType_t xQueueSend(
    QueueHandle_t xQueue,
    const void *pvItemToQueue,
    TickType_t xTicksToWait      // how long to block if queue is full
);
// Returns: pdPASS if item was sent, errQUEUE_FULL on timeout
```

**⭐ xQueueReceive**
```c
BaseType_t xQueueReceive(
    QueueHandle_t xQueue,
    void *pvBuffer,               // out: received item copied here
    TickType_t xTicksToWait
);
```

**⭐ xQueueSendFromISR / xQueueReceiveFromISR** — ISR-safe variants
```c
BaseType_t xQueueSendFromISR(
    QueueHandle_t xQueue,
    const void *pvItemToQueue,
    BaseType_t *pxHigherPriorityTaskWoken  // out: set pdTRUE if a higher-prio task was woken
);
```

**uxQueueMessagesWaiting** — check how many items are currently queued
```c
UBaseType_t uxQueueMessagesWaiting( QueueHandle_t xQueue );
```

---

## 3. Semaphores & Mutexes

**⭐ xSemaphoreCreateBinary**
```c
SemaphoreHandle_t xSemaphoreCreateBinary( void );
```

**xSemaphoreCreateCounting**
```c
SemaphoreHandle_t xSemaphoreCreateCounting(
    UBaseType_t uxMaxCount,
    UBaseType_t uxInitialCount
);
```

**⭐ xSemaphoreCreateMutex** — creates a mutex (supports priority inheritance)
```c
SemaphoreHandle_t xSemaphoreCreateMutex( void );
```

**⭐ xSemaphoreTake**
```c
BaseType_t xSemaphoreTake(
    SemaphoreHandle_t xSemaphore,
    TickType_t xTicksToWait
);
```

**⭐ xSemaphoreGive**
```c
BaseType_t xSemaphoreGive( SemaphoreHandle_t xSemaphore );
```

**⭐ xSemaphoreGiveFromISR / xSemaphoreTakeFromISR** — ISR-safe variants
```c
BaseType_t xSemaphoreGiveFromISR(
    SemaphoreHandle_t xSemaphore,
    BaseType_t *pxHigherPriorityTaskWoken
);
```

**xSemaphoreCreateRecursiveMutex / xSemaphoreTakeRecursive / xSemaphoreGiveRecursive**
— for a task that needs to re-lock a mutex it already holds (e.g., recursive function calls)
```c
SemaphoreHandle_t xSemaphoreCreateRecursiveMutex( void );
BaseType_t xSemaphoreTakeRecursive( SemaphoreHandle_t xMutex, TickType_t xTicksToWait );
BaseType_t xSemaphoreGiveRecursive( SemaphoreHandle_t xMutex );
```

---

## 4. Task Notifications (lightweight alternative to semaphores)

**⭐ xTaskNotifyGive / ulTaskNotifyTake**
```c
void vTaskNotifyGiveFromISR(
    TaskHandle_t xTaskToNotify,
    BaseType_t *pxHigherPriorityTaskWoken
);
uint32_t ulTaskNotifyTake(
    BaseType_t xClearCountOnExit,
    TickType_t xTicksToWait
);
```

**xTaskNotify / xTaskNotifyWait** — more general form, can pass a 32-bit value
```c
BaseType_t xTaskNotify(
    TaskHandle_t xTaskToNotify,
    uint32_t ulValue,
    eNotifyAction eAction   // e.g. eSetBits, eIncrement, eSetValueWithOverwrite
);
BaseType_t xTaskNotifyWait(
    uint32_t ulBitsToClearOnEntry,
    uint32_t ulBitsToClearOnExit,
    uint32_t *pulNotificationValue,
    TickType_t xTicksToWait
);
```

---

## 5. Event Groups

**xEventGroupCreate**
```c
EventGroupHandle_t xEventGroupCreate( void );
```

**xEventGroupSetBits / xEventGroupSetBitsFromISR**
```c
EventBits_t xEventGroupSetBits( EventGroupHandle_t xEventGroup, const EventBits_t uxBitsToSet );
```

**xEventGroupWaitBits** — block until specified bits are set (AND/OR configurable)
```c
EventBits_t xEventGroupWaitBits(
    EventGroupHandle_t xEventGroup,
    const EventBits_t uxBitsToWaitFor,
    const BaseType_t xClearOnExit,
    const BaseType_t xWaitForAllBits,   // pdTRUE = AND, pdFALSE = OR
    TickType_t xTicksToWait
);
```

---

## 6. Software Timers

**⭐ xTimerCreate**
```c
TimerHandle_t xTimerCreate(
    const char * const pcTimerName,
    const TickType_t xTimerPeriod,
    const UBaseType_t uxAutoReload,   // pdTRUE = periodic, pdFALSE = one-shot
    void * const pvTimerID,
    TimerCallbackFunction_t pxCallbackFunction
);
```

**⭐ xTimerStart / xTimerStop / xTimerReset**
```c
BaseType_t xTimerStart( TimerHandle_t xTimer, TickType_t xTicksToWait );
BaseType_t xTimerStop( TimerHandle_t xTimer, TickType_t xTicksToWait );
BaseType_t xTimerReset( TimerHandle_t xTimer, TickType_t xTicksToWait );
```

**xTimerChangePeriod**
```c
BaseType_t xTimerChangePeriod(
    TimerHandle_t xTimer,
    TickType_t xNewPeriod,
    TickType_t xTicksToWait
);
```

---

## 7. Critical Sections & Interrupt Control

**⭐ taskENTER_CRITICAL / taskEXIT_CRITICAL** — disable/enable interrupts (macros)
```c
taskENTER_CRITICAL();   // protect a very short shared-data access
// ... critical section ...
taskEXIT_CRITICAL();
```

**taskENTER_CRITICAL_FROM_ISR / taskEXIT_CRITICAL_FROM_ISR** — ISR-safe variants
```c
UBaseType_t uxSavedInterruptStatus = taskENTER_CRITICAL_FROM_ISR();
// ...
taskEXIT_CRITICAL_FROM_ISR( uxSavedInterruptStatus );
```

**vTaskSuspendAll / xTaskResumeAll** — suspend the scheduler (not interrupts) — lighter weight than a full critical section when you just need to stop task switches
```c
void vTaskSuspendAll( void );
BaseType_t xTaskResumeAll( void );
```

**portYIELD_FROM_ISR** — request a context switch at the end of an ISR (macro)
```c
portYIELD_FROM_ISR( xHigherPriorityTaskWoken );
```

---

## 8. Scheduler Control

**vTaskStartScheduler** — start the RTOS scheduler (never returns under normal operation)
```c
void vTaskStartScheduler( void );
```

**vTaskEndScheduler**
```c
void vTaskEndScheduler( void );
```

**vTaskSetTimeOutState / xTaskCheckForTimeOut** — used internally for building custom blocking APIs with correct timeout accounting

---

## 9. Memory Management

**pvPortMalloc / vPortFree** — FreeRTOS heap allocator (wraps whichever heap_x.c scheme is configured)
```c
void *pvPortMalloc( size_t xWantedSize );
void vPortFree( void *pv );
```

**xPortGetFreeHeapSize / xPortGetMinimumEverFreeHeapSize** — heap diagnostics (validation-relevant)
```c
size_t xPortGetFreeHeapSize( void );
size_t xPortGetMinimumEverFreeHeapSize( void );   // worst-case low water mark
```

---

## Quick Reference: Return / Type Conventions (say these if asked "what does BaseType_t mean")

| Type | Meaning |
|---|---|
| `BaseType_t` | Most efficient native integer type for the architecture — used for pass/fail returns (`pdPASS`, `pdFAIL`, `pdTRUE`, `pdFALSE`) |
| `TickType_t` | Represents time in RTOS ticks — width configurable (16 or 32-bit) |
| `UBaseType_t` | Unsigned version of BaseType_t — used for priorities, counts |
| `TaskHandle_t` | Opaque pointer identifying a task |
| `QueueHandle_t` / `SemaphoreHandle_t` / `TimerHandle_t` | Opaque handles to respective kernel objects |
| `pdMS_TO_TICKS(ms)` | Macro to convert milliseconds to ticks — use this instead of hardcoding tick counts |

---

## Interview Tip

Don't try to recite every signature from memory verbatim under pressure — that reads as rote memorization. Better approach: know **what each function does and why you'd choose it over an alternative** (e.g., "why xSemaphoreGiveFromISR instead of xSemaphoreGive" — because it's ISR-safe and reports whether a higher-priority task needs to run). If they want exact syntax, most interviewers are fine with you writing pseudo-signature and explaining parameters rather than compiler-perfect code, unless it's explicitly a live-coding round.
