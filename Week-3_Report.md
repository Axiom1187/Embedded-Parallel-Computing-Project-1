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
   ```
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

### Example (Bare-Metal Read)

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
```
### USART Output

```
char msg[50];
sprintf(msg, "Temp: %.2f C\r\n", TMP3_ReadTemp_Bare());
HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
```
---
## Part II - I²C with RTOS (CMSIS-RTOS v2)

### Configuration Notes

- Enable FreeRTOS → CMSIS-v2 API in CubeMX.
- Assign SysTick as the RTOS time base.
- Enable Newlib Reentrant if `printf` is used in multiple tasks.

---

### Implementation 

```
/* USER CODE BEGIN 0 */
float TMP3_ReadTemp(void) {
    uint8_t reg = TMP3_REG_TEMP, rx[2];
    if (HAL_I2C_Master_Transmit(&hi2c1, TMP3_I2C_ADDR, &reg, 1, 50) != HAL_OK) return NAN;
    if (HAL_I2C_Master_Receive(&hi2c1, TMP3_I2C_ADDR, rx, 2, 50) != HAL_OK) return NAN;

    int16_t raw = (rx[0] << 4) | (rx[1] >> 4);
    if (raw & 0x800) raw |= 0xF000;
    return raw * 0.0625f;
}
/* USER CODE END 0 */

void sensorReadTask(void *argument) {
    for (;;) {
        float t = TMP3_ReadTemp();
        if (!isnanf(t)) {
            char line[48];
            int n = snprintf(line, sizeof(line), "Temp: %.2f C\r\n", t);
            HAL_UART_Transmit(&huart2, (uint8_t*)line, n, 50);
        }
        osDelay(1000); // 1 Hz sampling
    }
}
```
---
### Behavior

- The RTOS scheduler executes `sensorReadTask` every 1 s.
- `osDelay(1000)` yields the CPU, preventing blocking.
- Data appears live on the serial terminal.

---

## How RTOS & USART Work Together

- Concurrency Model:
  The RTOS scheduler runs threads cooperatively; blocking calls (e.g., `HAL_I2C_Master_Receive`) use timeouts, allowing other threads to execute.

- UART Output:
  Short messages via `HAL_UART_Transmit` are fine for a single task. For multiple tasks that print, add a message queue or protect with a mutex.

- Tick Source:
  The SysTick timer now drives FreeRTOS. Use `osDelay()` instead of `HAL_Delay()` after the kernel starts.

---

## Requirements / Specifications / Verification  

| **Requirement** | **Specification** | **Verification** |
|------------------|------------------|------------------|
| **I²C Wiring** | PB8 (SCL), PB9 (SDA), 3.3 V logic | Visual check / oscilloscope |
| **I²C Configuration** | I²C1 Standard/Fast mode (100 – 400 kHz) | CubeMX `.ioc` review |
| **Task Periodicity** | `osDelay(1000)` → 1 Hz loop | UART output timing |
| **Data Conversion** | 12-bit two’s complement → °C | Compare with ambient |
| **USART Output** | 115200 baud, 8-N-1 (VCP) | PuTTY / Tera Term logs |
| **RTOS Integration** | CMSIS-RTOS v2 (`osThreadNew`, `osDelay`) | Debug trace / code review |
| **Reentrancy** | Newlib reentrant or mutex-guarded printing | Build settings check |

---

## Testing Procedure 

1. Flash firmware via CubeIDE
2. Open PuTTY at 115200 baud
3. Observe output
4. Warm sensor -> value rises; Cool sensor -> value lowers
5. Confirm steady 1Hz updates and no blocking behavior







