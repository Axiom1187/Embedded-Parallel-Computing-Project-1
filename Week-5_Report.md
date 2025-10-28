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



