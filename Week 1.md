# STM32 Microcontroller Project - Week 1

## 1. Project Description & Background
This project introduces embedded systems development using the **STM32F401RE Nucleo-64** microcontroller board. The objective is to build familiarity with STM32 system.

- Installing the STM32 software toolchain (CubeIDE, CubeMX, CubeProgrammer).
- Creating and running an application to blink a LED (LD2).
- Configuring UART communication to send messages from the microcontroller to a host PC.

The STM32F401RE is based on the Cortex-M4 core, mainly used in academic and industrial applications for real-time embedded systems. Gaining proficiency with this platform establishes skills for more advanced tasks such as peripheral interfacing, RTOS integration, and system-level design.

---

## 2. System Architecture

- **Host PC**: Runs STM32CubeIDE for coding, CubeMX for configuration, CubeProgrammer for flashing, and a serial terminal for monitoring UART output.
- **STM32F401RE MCU**: Executes firmware, toggles the LED via GPIO, and transmits messages via UART.
- **Virtual COM Port (ST-LINK)**: Handles UART-to-USB communication.

┌─────────────────────────────┐
│ Host PC                     │
│ ─────────────────────────── │
│ STM32CubeIDE (Code Dev)     │
│ STM32CubeMX (Config)        │
│ STM32CubeProgrammer (Flash) │
│ Serial Terminal (UART COM)  │
└──────────────┬──────────────┘
      USB (ST-LINK/V2-1)
┌──────────────▼───────────────┐
│ STM32F401RE Nucleo-64        │
│ ───────────────────────────  │
│ GPIO (PA5) → LD2 (Blink)     │
│ USART2 (PA2/PA3) → ST-LINK   │
│ ST-LINK/V2-1 Debugger        │
└──────────────┬───────────────┘
           Power/USB
          ┌───▼───┐
          │ LED   │
          └───────┘


---

## 3. Requirements, Specifications, Verification

| **Requirement**               | **Specification (STM32F401RE)**               | **Verification Method**                                      |
|-------------------------------|-----------------------------------------------|--------------------------------------------------------------|
| Development environment setup | Install STM32CubeIDE, CubeMX, CubeProgrammer  | Verify IDE opens, CubeMX generates project, board detected   |
| LED Blink functionality       | Blink **LD2 (PA5)** every 1s using HAL        | Observe on-board LED blinking at 1 Hz                        |
| Board–PC connection           | STM32F401RE enumerates as ST-LINK/V2-1 via USB| Confirm detection in CubeProgrammer / Device Manager         |
| Firmware deployment           | Compile, flash, and run code on STM32F401RE   | Program executes without errors                              |
| Serial output                 | Use **USART2 (PA2=TX, PA3=RX)** via ST-LINK   | Open terminal (115200 baud, 8-N-1) and confirm messages      |
| Debugging support             | Step execution, watch variables via ST-LINK   | Confirm in CubeIDE debug perspective                         |

---

## 4. Selected Hardware, APIs, and Tools

### Hardware
- **STM32F401RE Nucleo-64** development board
  - MCU: ARM Cortex-M4 @ 84 MHz
  - 512 KB Flash, 96 KB SRAM
  - On-board LED: **LD2 (Pin PA5)**
  -
