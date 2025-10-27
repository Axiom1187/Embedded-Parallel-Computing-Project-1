# Project 1 — Week 3 Report  
**Board:** STM32F401RE Nucleo-64  
**RTOS Layer:** CMSIS-RTOS v2 (FreeRTOS Kernel)  
**Peripherals:** I²C Sensor, USART2 (115200 baud), GPIO (LD2), ST-LINK VCP  

---

## 1️⃣ Project Description & Background  

This week extends Project 1 by integrating a **real-time operating system (RTOS)** and adding **I²C sensor communication** with serial output.  
The project demonstrates **parallel tasking** and hardware abstraction using **CMSIS-RTOS v2**, which serves as a wrapper around the FreeRTOS kernel.

### 🎯 Objectives  
- Implement **I²C communication** with an external temperature sensor.  
- Use **USART2** to print sensor data to a terminal on the host PC.  
- Transition from bare-metal polling to **multi-tasking using RTOS threads**.  
- Verify periodic task execution using the **CMSIS-RTOS v2 API**.

### 💡 Why CMSIS-RTOS v2?  
CMSIS-RTOS v2 provides a hardware-agnostic API layer standardized by ARM.  
STM32Cube integrates this with a **glue layer** (`cmsis_os2.c/h`) that translates CMSIS calls (e.g., `osThreadNew`, `osDelay`) into equivalent FreeRTOS primitives.  
This enables portability, modular design, and cleaner task scheduling compared to direct HAL-based loops.

---

## 2️⃣ System Architecture  

### 🧩 Overview  

