# Project 1 — Week 3 Report  
**Board:** STM32F401RE Nucleo-64  
**RTOS Layer:** CMSIS-RTOS v2 (FreeRTOS Kernel)  
**Peripherals:** I²C Sensor, USART2 (115200 baud), GPIO (LD2), ST-LINK VCP  

---

## Project Description & Background  

This week extends Project 1 by integrating a **real-time operating system (RTOS)** and adding **I²C sensor communication** with serial output.  
The project demonstrates **parallel tasking** and hardware abstraction using **CMSIS-RTOS v2**, which serves as a wrapper around the FreeRTOS kernel.

### Objectives  
- Implement **I²C communication** with an external temperature sensor.  
- Use **USART2** to print sensor data to a terminal on the host PC.  
- Transition from bare-metal polling to **multi-tasking using RTOS threads**.  
- Verify periodic task execution using the **CMSIS-RTOS v2 API**.

### Why CMSIS-RTOS v2?  
CMSIS-RTOS v2 provides a hardware-agnostic API layer standardized by ARM.  
STM32Cube integrates this with a **glue layer** (`cmsis_os2.c/h`) that translates CMSIS calls (e.g., `osThreadNew`, `osDelay`) into equivalent FreeRTOS primitives.  
This enables portability, modular design, and cleaner task scheduling compared to direct HAL-based loops.

---

## System Architecture  

### Overview  

┌────────────────────────────┐
│ Host PC                    │
│ • PuTTY                    │
│ • STM32CubeProgrammer      │
│ • STM32CubeIDE (debug)     │
└───────────────┬────────────┘
│ USB (ST-LINK V2-1, VCP)
┌───────────────▼────────────┐
│ STM32F401RE Nucleo         │
│ ┌───────────────────────┐  │
│ │ CMSIS-RTOS v2 (FreeRTOS) │
│ │ • sensorReadTask         │
│ └───────────────────────┘  │
│ Peripherals:               │
│ • I²C1 → Sensor (SCL/SDA)  │
│ • USART2 → PC (printf)     │
│ • GPIO → LD2 (Heartbeat)   │
└────────────────────────────┘

### Task Scheduling  

| **Task** | **Description** | **Priority** | **Period** |
|-----------|----------------|---------------|-------------|
| `sensorReadTask` | Reads temperature sensor, converts data, prints over USART | Normal | 1 s |
| `defaultTask` | Cube-generated idle / test thread | Low | – |

---

## Code Walkthrough  

### Initialization  
1. **Kernel Startup**
   ```c
   osKernelInitialize();
   osThreadNew(sensorReadTask, NULL, &sensorReadTask_attributes);
   osKernelStart();

