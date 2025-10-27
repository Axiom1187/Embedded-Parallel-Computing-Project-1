# Project 1 — Week 3 Report  
**Board:** STM32F401RE Nucleo-64  
**RTOS Layer:** CMSIS-RTOS v2 (FreeRTOS Kernel)  
**Peripherals:** I²C Sensor, USART2 (115200 baud), GPIO (LD2), ST-LINK VCP  

---

## Project Description & Background  

This week extends Project 1 by integrating a real-time operating system (RTOS) and adding I²C sensor communication with serial output.  
The project demonstrates parallel tasking and hardware abstraction using CMSIS-RTOS v2, which serves as a wrapper around the FreeRTOS kernel.

### Objectives  
- Implement I²C communication with an external temperature sensor.  
- Use USART2 to print sensor data to a terminal on the host PC.  
- Transition from bare-metal polling to multi-tasking using RTOS threads.  
- Verify periodic task execution using the CMSIS-RTOS v2 API.

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

## Starts the RTOS kernel and creates the periodic sensor task.

---

### ⚙️ USART2 Configuration

- **Baud Rate:** 115200 bps  
- **Word Length:** 8 bits  
- **Stop Bits:** 1  
- **Parity:** None  
- **Flow Control:** None  
- **Oversampling:** ×16  
- **Purpose:** Used for serial logging via **ST-LINK Virtual COM Port**.

---

### 🔌 I²C Pins

- PB8 = SCL, PB9 = SDA 
- Alternate Function 4, Open-Drain, Pull-Up enabled.  

---

## Part I — I²C Sensor Read (Bare Metal)

### CubeMX Setup

- Enable **I²C1** under *Connectivity → I²C1* (Mode = I²C).  
- Assign pins **PB8 (SCL)** and **PB9 (SDA)**.  
- Verify pull-ups (**4.7 kΩ typical**).  
- Generate code → confirm `MX_I2C1_Init()` is called in `main()`.

---

### 🧮 Example (Bare-Metal Read)

```c
#define TMP3_I2C_ADDR   (0x48 << 1)
#define TMP3_REG_TEMP   0x00

float TMP3_ReadTemp_Bare(void) {
    uint8_t rx[2];
    uint8_t reg = TMP3_REG_TEMP;

    HAL_I2C_Master_Transmit(&hi2c1, TMP3_I2C_ADDR, &reg, 1, HAL_MAX_DELAY);
    HAL_I2C_Master_Receive(&hi2c1, TMP3_I2C_ADDR, rx, 2, HAL_MAX_DELAY);

    int16_t raw = (rx[0] << 4) | (rx[1] >> 4);
    if (raw & 0x800) raw |= 0xF000;
    return raw * 0.0625f;
}



