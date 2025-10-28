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
QueueHandle_t sensorQueue;  
QueueHandle_t actuatorQueue;
```
- `sensorQueue` transfers sensor data from Sensor Task -> Control Task
- `actuatorQueue` transfers control triggers from Control Task -> Actuator Task

### Data Structures

```c
typedef struct {
  float temperature;  
  uint8_t sensorStatus;
} SensorData_t;

typedef struct {
  uint8_t triggerState;
} ActuatorData_t;
```
### Queue Creation

```
sensorQueue  = xQueueCreate(5, sizeof(SensorData_t));  
actuatorQueue = xQueueCreate(5, sizeof(ActuatorData_t));
```
Queues are allocated before the scheduler starts in `main()` and checking for non-NULL return values.

---

## Task Creation and Scheduler Start

```c
xTaskCreate(SensorTask, "SensorTask", 256, NULL, 2, NULL);  
xTaskCreate(ControlTask, "ControlTask", 256, NULL, 1, NULL);  
xTaskCreate(ActuatorTask, "ActuatorTask", 256, NULL, 1, NULL);  
xTaskCreate(LoggingTask, "LoggingTask", 256, NULL, 1, NULL);

vTaskStartScheduler();  // Starts the FreeRTOS kernel
```
Each task is assigned a stack and priority. If no idle hook is defined, FreeRTOS creates an implicit Idle Task for background operation.

---

## Task Implementations

### Sensor Task 

Reads the TMP102 sensor via I²C and sends data to `ControlTask` through `sensorQueue`

```c
void SensorTask(void *argument) {
  SensorData_t sensorData;
  for (;;) {
    uint8_t data[2];
    HAL_I2C_Master_Transmit(&hi2c1, TMP102_ADDR, &REG_TEMP, 1, 100);
    HAL_I2C_Master_Receive(&hi2c1, TMP102_ADDR, data, 2, 100);
    int16_t val = ((int16_t)data[0] << 4) | (data[1] >> 4);
    if (val > 0x7FF) val |= 0xF000;
    sensorData.temperature = val * 0.0625f;
    sensorData.sensorStatus = 1;
    xQueueSend(sensorQueue, &sensorData, portMAX_DELAY);
    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}
```

### Control Task

Receives sensor data, applies control logic, and sends trigger to `ActuatorTask`.

```c
void ControlTask(void *argument) {
  SensorData_t sensorData;
  ActuatorData_t actData;
  for (;;) {
    if (xQueueReceive(sensorQueue, &sensorData, portMAX_DELAY) == pdPASS) {
      actData.triggerState = (sensorData.temperature > 27.0f);
      xQueueSend(actuatorQueue, &actData, portMAX_DELAY);
    }
    vTaskDelay(pdMS_TO_TICKS(500));
  }
}
```
- Uses `xQueueReceive()` to block until new data arrives
- Sends binary trigger to Actuator Task

### Actuator Task

Toggles LED or PWM output based on received trigger

```c
void ActuatorTask(void *argument) {
  ActuatorData_t actData;
  for (;;) {
    if (xQueueReceive(actuatorQueue, &actData, portMAX_DELAY) == pdPASS) {
      if (actData.triggerState)
        HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_SET);
      else
        HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
    }
  }
}
```
This acts as the final stage of the data pipeline (Sensor -> Control -> Actuator).

### Logging Task

Periodically prints queue contents over UART for debugging.

```c
void LoggingTask(void *argument) {
  for (;;) {
    char buf[64];
    snprintf(buf, sizeof(buf), "Temp = %.2f °C\r\n", latestTemp);
    HAL_UART_Transmit(&huart2, (uint8_t*)buf, strlen(buf), 50);
    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}
```
Allows verification of sensor values and queue operation via serial monitor.

---

## Inter-Task Communication Flow

SensorTask → (sensorQueue) → ControlTask → (actuatorQueue) → ActuatorTask
                                                                    │  
                                                                    └── LoggingTask (UART monitor)












