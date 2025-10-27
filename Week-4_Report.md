# Project 1 — Week 4 Report  
**Board:** STM32F401RE Nucleo-64  
**RTOS Layer:** CMSIS-RTOS v2 (FreeRTOS Kernel)  
**Peripherals:** I²C Sensor (TMP102), USART2 (115200 baud), GPIO (LED LD2)

---

## Overview  

This week’s milestone extends Week 3 by adding multiple RTOS tasks and demonstrating context switching in FreeRTOS.  

The project now contains three parallel tasks:

| Task | Purpose | Priority |
|-------|-----------|-----------|
| **Sensor Task** | Reads temperature from TMP102 via I²C and updates a shared global variable. | Above Normal |
| **Control Task** | Reads the global temperature value and toggles an LED when a set point is exceeded. | Normal |
| **Default Task** | Idle loop used for timing and scheduling baseline work. | Normal |

Later weeks will add an Actuator Task and a Logging Task to reach four concurrent threads.

---

## Project Setup Summary  

1. Week 3 project was cloned to `TMP3task_and_ControlTask` workspace folder.  
2. In CubeMX, FreeRTOS → Tasks and Queues was opened and a new task named `ControlTask` was added with priority lower than `SensorTask`.  
3. Pin PA5 (LED LD2) was enabled as GPIO Output to serve as a trigger indicator.  
4. USART2 (TX/RX) was verified active for serial logging.  
5. Code was generated and compiled in STM32CubeIDE.

---

## Code Structure and Task Explanation  

### Sensor Task (`StartTask02`) :contentReference[oaicite:0]{index=0}

- Configures I²C1 to communicate with TMP102 (0x48 << 1).  
- Requests the temperature register and reads two bytes.  
- Combines and sign-extends the 12-bit reading to a signed 16-bit value.  
- Converts raw data to Celsius (`value × 0.0625`).  
- Stores the result in the global variable `g_temperature`.  
- Transmits the value to the serial terminal over USART2 for debugging.  
- Repeats every 1 s via `osDelay(1000)`.

```
g_temperature = val * 0.0625f;
sprintf((char*)buf, "%.2f C\r\n", g_temperature);
HAL_UART_Transmit(&huart2, buf, strlen((char*)buf), HAL_MAX_DELAY);
```
---
### Control Task (`StartTask03`) 

- Reads the shared global temperature g_temperature.
- Compares it against `TEMP_SETPOINT = 27.0 °C`.
- If the temperature exceeds the set point, turns LED LD2 ON; otherwise OFF.
- Runs twice per second (`osDelay(500)`), showing real-time trigger behavior.

```
if (g_temperature > TEMP_SETPOINT) {
    HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_SET);
} else {
    HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
}
```
This simulates an actuator response loop and provides a visual indicator for context switches between tasks.
---
### Default Task (`StartDefaultTask`) 

A low-priority loop that executes osDelay(1) continuously. It represents the idle thread and confirms scheduler operation even when other tasks sleep.
---
## Context Switch Visualization

### Trace Hook Macro in FreeRTOSConfig.h

```
#define traceTASK_SWITCHED_IN() traceTaskSwitch()
```
Whenever the FreeRTOS scheduler switches to a different task, it calls the user-defined `traceTaskSwitch()` function in `main.c`.

### Trace Function Implementation
```
void traceTaskSwitch(void)
{
    static uint32_t counter = 0;
    counter++;

    if (counter % 1000 == 0) {  // print every 1000th switch
        uint8_t buf1[50];
        snprintf((char*)buf1, sizeof(buf1), "Current task: %s\r\n",
                 pcTaskGetName(NULL));
        HAL_UART_Transmit(&huart2, buf1, strlen((char*)buf1), HAL_MAX_DELAY);
    }
}
```
- `HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5)` can also be used to toggle LD2 for visual feedback.
- This mechanism confirms task context switches in a single-core MCU like the STM32F401RE.

Messages such as `Current task: SensorTask` or `Current task: ControlTask` appear in the serial console.
---
## Inter-Task Communication

- A shared global variable `g_temperature` is declared in the Private Variables section of `main.c`.
- `SensorTask` updates this variable; `ControlTask` reads it.
- This simple approach works since both tasks are synchronized by Delays and FreeRTOS’s preemptive scheduler.
- For future projects, this should be replaced by thread-safe mechanisms like queues or mutexes.
---
## Task Scheduling and Priorities

- The FreeRTOS scheduler uses a preemptive round-robin algorithm controlled by the tick interrupt (`configTICK_RATE_HZ = 1000` Hz in `FreeRTOSConfig.h`).
- `SensorTask` has higher priority (`osPriorityAboveNormal`) than `ControlTask (osPriorityNormal)`.
- When SensorTask runs, it reads the sensor and delays for 1 s; during that time the scheduler runs `ControlTask` and `defaultTask`.
- This creates cooperative multitasking on a single core, simulating parallelism through rapid context switches.
---
## FreeRTOSConfig.h Highlights  

The `FreeRTOSConfig.h` file defines system-wide configuration macros that determine task scheduling, timing, and kernel behavior.  
Below are the key parameters used in this project:

| **Parameter** | **Value** | **Effect** |
|----------------|------------|-------------|
| `configUSE_PREEMPTION` | `1` | Enables preemptive multitasking, allowing higher-priority tasks to interrupt lower-priority ones. |
| `configTICK_RATE_HZ` | `1000` | Sets the system tick to 1 ms, giving precise task timing and delays. |
| `configMAX_PRIORITIES` | `7` | Allows fine-grained task priority control across multiple threads. |
| `configTOTAL_HEAP_SIZE` | `15360 bytes` | Defines the heap memory pool for dynamic task and object allocation. |
| `configASSERT(x)` | Custom halt if assertion fails | Stops the system for debugging when a kernel-level assertion fails. |
| `traceTASK_SWITCHED_IN()` | Linked to `traceTaskSwitch()` | Calls a **user-defined trace function** on every context switch for task monitoring. |

These settings ensure real-time responsiveness and help demonstrate FreeRTOS task management concepts.

---
## Testing and Verification










