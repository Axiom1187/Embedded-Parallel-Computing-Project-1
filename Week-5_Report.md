# Project 1 — Week 5 Report  
**Board:** STM32F401RE Nucleo-64  
**RTOS Layer:** FreeRTOS (Native API)  
**Peripherals:** I²C Sensor (TMP102), USART2 (115200 baud), GPIO (LED LD2), Timer PWM  

---

## Introduction  

This week’s milestone replaces the CMSIS-RTOS v2 wrapper used in Weeks 3-4 with the native FreeRTOS API.  
Four concurrent tasks are now defined:

| **Task** | **Purpose** |
|-----------|-------------|
| **Sensor Task** | Reads temperature data from TMP102 sensor via I²C and sends it to `Control Task` through a queue. |
| **Control Task** | Receives sensor data, applies control logic, and sends actuator triggers to `Actuator Task`. |
| **Actuator Task** | Drives the GPIO/PWM output to control a motor or indicator based on the control trigger. |
| **Logging Task** | Logs calibrated sensor values over UART or to file storage for debugging and verification. |

Key learning outcomes:  
- Replace CMSIS functions (`osThreadNew`, `osDelay`) with FreeRTOS native API (`xTaskCreate`, `vTaskDelay`).  
- Implement inter-task communication using queues (`xQueueCreate`, `xQueueSend`, `xQueueReceive`).  
- Expand to four tasks executing concurrently with predictable scheduling.

---

## 2️⃣ System Architecture  

┌──────────────────────────────┐

│ Host PC │

│ • PuTTY / Tera Term console │

│ • STM32CubeIDE (Debug) │

└───────────────┬──────────────┘

│ USB (VCP)

┌───────────────▼──────────────┐

│ STM32F401RE Nucleo Board │

│ ─────────────────────────── │

│ FreeRTOS Scheduler (ticks) │

│ • Sensor Task → I²C TMP102 │

│ • Control Task → Logic + Trigger │

│ • Actuator Task → PWM Output │

│ • Logging Task → UART Logs │

│ Queues: Sensor → Control → Actuator │

│ LD2 (LED) = Visual indicator │

└──────────────────────────────┘

---

## Transition from CMSIS to Native FreeRTOS  

| **Feature** | **CMSIS-RTOS v2** | **FreeRTOS Native API** |
|--------------|------------------|---------------------------|
| Task creation | `osThreadNew()` | `xTaskCreate()` |
| Queue creation | `osMessageQueueNew()` | `xQueueCreate()` |
| Send / Receive | `osMessageQueuePut()` / `osMessageQueueGet()` | `xQueueSend()` / `xQueueReceive()` |
| Delay | `osDelay(ms)` | `vTaskDelay(pdMS_TO_TICKS(ms))` |
| Scheduler start | `osKernelStart()` | `vTaskStartScheduler()` |

This change allows direct use of the FreeRTOS kernel without the overhead of CMSIS abstraction.

---

## Queue Setup and Data Structures  

### Queue Handles  
```c
// Queue handles are kernel objects (pointers) that FreeRTOS uses to manage queues.
// We define two queues:
// 1) sensorQueue: SensorTask → ControlTask (passes measured sensor data)
// 2) actuatorQueue: ControlTask → ActuatorTask (passes on/off trigger or control value)
QueueHandle_t sensorQueue;
QueueHandle_t actuatorQueue;

```
- `sensorQueue` transfers sensor data from Sensor Task -> Control Task
- `actuatorQueue` transfers control triggers from Control Task -> Actuator Task

### Data Structures

```c
// Pack the sensor signal(s) and status into a struct so the queue carries
// a single, well-defined message type (safer and extensible than raw floats).
typedef struct {
  float   temperature;   // degrees Celsius (calibrated in task)
  uint8_t sensorStatus;  // e.g., 1=OK, 0=error/invalid reading
} SensorData_t;

// Minimal actuator command message—can be extended later (e.g., PWM duty, mode).
typedef struct {
  uint8_t triggerState;  // 1 = ON / active trigger, 0 = OFF
} ActuatorData_t;

```
### Queue Creation

```
// Create the queues before starting the scheduler.
// Length = 5 messages; Item size = sizeof(message struct).
// If creation returns NULL, heap is insufficient or parameters invalid—check configTOTAL_HEAP_SIZE.
sensorQueue   = xQueueCreate(5, sizeof(SensorData_t));
actuatorQueue = xQueueCreate(5, sizeof(ActuatorData_t));

```
Queues are allocated before the scheduler starts in `main()` and checking for non-NULL return values.

---

## Task Creation and Scheduler Start

```c
// Create four application tasks. Stack sizes are examples—tune via run-time stack high-water marks.
// Priorities: higher number = higher priority (relative scheme).
xTaskCreate(SensorTask,   "SensorTask",   256, NULL, 2, NULL);  // producer of SensorData_t
xTaskCreate(ControlTask,  "ControlTask",  256, NULL, 1, NULL);  // consumes sensor, produces trigger
xTaskCreate(ActuatorTask, "ActuatorTask", 256, NULL, 1, NULL);  // consumes trigger → drives output
xTaskCreate(LoggingTask,  "LoggingTask",  256, NULL, 1, NULL);  // optional, UART prints / storage

// Hand control to the RTOS. If it returns, something failed (e.g., not enough heap for Idle/Timer tasks).
vTaskStartScheduler();

```
Each task is assigned a stack and priority. If no idle hook is defined, FreeRTOS creates an implicit Idle Task for background operation.

---

## Task Implementations

### Sensor Task 

Reads the TMP102 sensor via I²C and sends data to `ControlTask` through `sensorQueue`

```c
// Reads the TMP102 every 1 second and sends a message to ControlTask via sensorQueue.
void SensorTask(void *argument) {
  SensorData_t sensorData;                 // local message instance (lives on task stack)
  for (;;) {
    uint8_t data[2];                       // raw sensor bytes
    // Select temperature register on TMP102 (write register pointer)
    HAL_I2C_Master_Transmit(&hi2c1, TMP102_ADDR, &REG_TEMP, 1, 100);
    // Read 2 bytes from temperature register
    HAL_I2C_Master_Receive(&hi2c1, TMP102_ADDR, data, 2, 100);

    // Convert 12-bit two's complement to signed 16-bit:
    // - Data format: [MSB:8 bits][LSB: upper 4 bits valid, lower 4 bits fractional alignment]
    int16_t raw = ((int16_t)data[0] << 4) | (data[1] >> 4);
    if (raw > 0x7FF) raw |= 0xF000;       // sign-extend negative values

    // Scale to °C (TMP102 LSB = 0.0625°C)
    sensorData.temperature = raw * 0.0625f;
    sensorData.sensorStatus = 1;          // mark sample valid (could set to 0 if an I2C HAL call fails)

    // Send to ControlTask. portMAX_DELAY blocks indefinitely until space is available
    // (backpressure ensures the producer won't outrun the consumer).
    xQueueSend(sensorQueue, &sensorData, portMAX_DELAY);

    // Sleep 1 second using RTOS-aware delay (does not block other tasks).
    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}

```

### Control Task

Receives sensor data, applies control logic, and sends trigger to `ActuatorTask`.

```c
// Receives latest sensor data, applies threshold logic, and forwards a trigger to ActuatorTask.
void ControlTask(void *argument) {
  SensorData_t   sensorData;   // inbound message buffer
  ActuatorData_t actData;      // outbound message buffer

  for (;;) {
    // Block until new sensor data arrives (no busy-waiting).
    if (xQueueReceive(sensorQueue, &sensorData, portMAX_DELAY) == pdPASS) {
      // Simple threshold control: ON if temperature > 27 °C, else OFF.
      // Replace with your control algo (hysteresis, PID, etc.) as needed.
      actData.triggerState = (sensorData.temperature > 27.0f) ? 1 : 0;

      // Send actuation command to the next stage.
      xQueueSend(actuatorQueue, &actData, portMAX_DELAY);
    }

    // Optional pacing to avoid saturating CPU if you later make the receive non-blocking.
    vTaskDelay(pdMS_TO_TICKS(500));
  }
}

```
- Uses `xQueueReceive()` to block until new data arrives
- Sends binary trigger to Actuator Task

### Actuator Task

Toggles LED or PWM output based on received trigger

```c
// Consumes control triggers and drives the hardware (LED, PWM, motor, etc.).
void ActuatorTask(void *argument) {
  ActuatorData_t actData;

  for (;;) {
    // Block until a new trigger command arrives
    if (xQueueReceive(actuatorQueue, &actData, portMAX_DELAY) == pdPASS) {
      if (actData.triggerState) {
        // Example: turn LD2 ON (replace with PWM update or motor step sequence)
        HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_SET);
      } else {
        // Example: turn LD2 OFF
        HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
      }
    }
    // No delay here—this task is event-driven by the queue.
  }
}

```
This acts as the final stage of the data pipeline (Sensor -> Control -> Actuator).

### Logging Task

Periodically prints queue contents over UART for debugging.

```c
// Periodically logs the most recent values/status for verification via UART.
// You can make this subscribe to queues too, but simple time-based reads are fine for diagnostics.
void LoggingTask(void *argument) {
  for (;;) {
    char buf[64];
    // In a full design, keep a thread-safe "latest" snapshot or receive from a log queue
    // to avoid reading globals. For quick checks, this is acceptable.
    extern float latestTemp; // example: a snapshot updated elsewhere (or pass via queue)
    int n = snprintf(buf, sizeof(buf), "Temp = %.2f °C\r\n", latestTemp);
    HAL_UART_Transmit(&huart2, (uint8_t*)buf, (uint16_t)n, 50);

    // Pace the logging to avoid flooding the serial terminal.
    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}

```
Allows verification of sensor values and queue operation via serial monitor.

---

## Inter-Task Communication Flow

```scss
SensorTask → (sensorQueue) → ControlTask → (actuatorQueue) → ActuatorTask
                                                                    │  
                                                                    └── LoggingTask (UART monitor)
```
Each queue isolates producer/consumer logic so tasks operate independently and non-blocking

---

## Testing Procedure

1. Building and flash the project in the STM32CubeIDE
2. Open PuTTY at 115200 baud
3. Observe live temperature values printed by Logging Task
4. Warm the TMP102 sensor -> LED LD2 activates when temperature > 27 °C.
5. Cool sensor -> LED turns off.
6. Confirm timing and smooth context switching between all four tasks.

---

## Benefits of Using FreeRTOS Queues

- Eliminates shared global variables and race conditions
- Built-in blocking mechanisms simplify synchronization
- Improves code scalability and modularity for future sensors and actuators
- Provides deterministic real-time data transfer between tasks.

---

## Key Takeaways

- Migrated successfully from CMSIS-RTOS v2 to natice FreeRTOS API
- Implemented four tasks with two data queues to enable full RTOS commuication
- Demonstrated end-to-end real-time control loop (sensir












