# STM32 Task 1 – Bare-Metal UART Authentication

## Target MCU

- **MCU:** STM32F103C8T6 (STM32 Blue Pill)
- **CPU core:** ARM Cortex-M3
- **Build:** ARM GNU Toolchain (`arm-none-eabi-gcc`)
- **Programming style:** Bare-metal register access, custom startup code and linker script; no HAL/LL APIs and no CMSIS startup/system files.

## Pin Mapping

| Function | STM32 Pin | Register configuration / notes |
|---|---|---|
| User LED | **PC13** | GPIOC output, push-pull, 2 MHz; LED is **active-low** on the Blue Pill |
| Button | **None implemented in Task 1** | No push-button is used by this task |
| UART TX | **PA9 / USART1_TX** | Alternate-function push-pull, 50 MHz |
| UART RX | **PA10 / USART1_RX** | Floating input |

### Wokwi Serial Monitor Mapping

The Wokwi diagram connects the simulated serial monitor to USART1 as follows:

- Serial Monitor **TX → PA10 (USART1 RX)**
- Serial Monitor **RX → PA9 (USART1 TX)**

Serial settings:

- Baud rate: **115200**
- Data bits: **8**
- Parity: **None**
- Stop bits: **1**
- Line ending: **LF (`\\n`)**

## Register Base Addresses and Offsets Used

### RCC

- **RCC base:** `0x40021000`
- **APB2 peripheral clock enable register:** `RCC_APB2ENR`
- **Address:** `0x40021018`
- **Offset:** `0x18`
- Bit 2 (`IOPAEN`) enables GPIOA.
- Bit 4 (`IOPCEN`) enables GPIOC.
- Bit 14 (`USART1EN`) enables USART1.

### GPIOA

- **GPIOA base:** `0x40010800`
- **CRH:** `0x40010804`
- **Offset:** `0x04`
- PA9 and PA10 configuration is stored in the CRH fields.

### GPIOC

- **GPIOC base:** `0x40011000`
- **CRH:** `0x40011004`
- **Offset:** `0x04`
- PC13 configuration uses CRH bits `[23:20]`.
- **ODR:** `0x4001100C`
- **Offset:** `0x0C`
- PC13 is controlled through ODR bit 13.

### USART1

- **USART1 base:** `0x40013800`

| Register | Address | Offset | Purpose |
|---|---:|---:|---|
| `USART1_SR` | `0x40013800` | `0x00` | Status register; RXNE/TXE polling |
| `USART1_DR` | `0x40013804` | `0x04` | Data register; receive/transmit byte |
| `USART1_BRR` | `0x40013808` | `0x08` | Baud-rate register |
| `USART1_CR1` | `0x4001380C` | `0x0C` | USART enable, TX enable, RX enable |
| `USART1_CR2` | `0x40013810` | `0x10` | Frame configuration; default 1 stop bit |
| `USART1_CR3` | `0x40013814` | `0x14` | Additional USART features; default configuration |

## Clock and UART Configuration

The startup code does **not** configure the RCC/PLL. The STM32 starts from the internal HSI clock, so:

- HSI = **8 MHz**
- PCLK2 = **8 MHz**
- USART1 baud-rate register is set to `BRR = 0x0045`, giving approximately **115200 baud**.

USART1 is configured for **8-N-1** polling communication.

## Boot Flow: Reset to `main()`

1. **Reset occurs.** The Cortex-M3 reads the first two words of the vector table from Flash.
2. The first word is `_estack`, which provides the **initial stack pointer**.
3. The second word is `Reset_Handler`, so execution jumps to the custom reset handler in `startup.s`.
4. `Reset_Handler` copies the initialized **`.data` section from Flash to RAM** using `_sidata`, `_sdata`, and `_edata`.
5. It then clears the **`.bss` section in RAM** from `_sbss` to `_ebss`, setting it to zero.
6. After RAM initialization, the reset handler executes `bl main` and enters the C program.
7. `main()` initializes **PC13 LED GPIO** and **USART1**, then continuously receives UART commands and processes authentication requests.
8. `main()` is expected to run forever. If it ever returns, the startup code enters an infinite `hang` loop.

## Authentication Protocol

The firmware expects:

```text
AUTH:<8 hexadecimal characters>\n
```

For a valid command, it:

1. Parses the 32-bit challenge.
2. Appends the shared 32-bit secret key in big-endian byte order.
3. Calculates standard reflected CRC32 using polynomial `0xEDB88320`.
4. Returns:

```text
RESP:<8 hexadecimal characters>\n
```

5. Toggles the PC13 LED after a successful authentication response.

Invalid or malformed commands return:

```text
ERR\n
```

## Shared Secret

The firmware and host test use the same 32-bit secret key:

```c
0x1A2B3C4D
```

## Build Commands

From the Task1 project directory:

```bash
make
```

Clean generated object and firmware files:

```bash
make clean
```

A successful build generates:

- `task1.elf`
- `task1.bin`

## Project Files

```text
Task1/
├── Makefile
├── linker.ld
├── diagram.json
├── wokwi.toml
├── task1.elf
├── task1.bin
└── src/
    ├── main.c
    ├── uart.c
    ├── uart.h
    ├── crc32.c
    ├── crc32.h
    ├── startup.s
    ├── host_test.py
    └── send_x.py
```

## Notes

- No hardware button is required or used in Task 1.
- UART uses polling rather than interrupts.
- The custom startup file supplies the vector table and reset handler.
- The linker script places the vector table and program code in Flash and RAM data in the correct STM32 memory regions.
