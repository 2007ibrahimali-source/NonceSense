# STM32 Task 0 — Bare-Metal LED Blink

## 1. Project Overview

This project implements a minimal bare-metal firmware for an STM32 Blue Pill using direct register access.

The program:

- Starts from the Cortex-M3 reset vector.
- Initializes the `.data` and `.bss` sections in RAM.
- Enables GPIOC.
- Configures the onboard LED on PC13 as a push-pull output.
- Blinks the LED continuously with an approximately 500 ms delay between toggles.
- Uses no HAL, LL, or CMSIS startup/system files.

The project is designed to build with the ARM GNU toolchain and can be simulated in Wokwi.

---

## 2. Target MCU

**MCU:** STM32F103C8T6  
**Core:** ARM Cortex-M3  
**Board:** STM32 Blue Pill  
**Flash:** 64 KB used by the linker script  
**RAM:** 20 KB used by the linker script

The linker script places:

- FLASH at `0x08000000`
- RAM at `0x20000000`

---

## 3. Pin Mapping

| Function | STM32 Pin | Notes |
|---|---|---|
| Onboard LED | PC13 | Active-low LED |
| Button | Not used | No external/onboard button is used by this task |
| UART TX | Not used | Task 0 has no UART |
| UART RX | Not used | Task 0 has no UART |

### LED behavior

The LED is **active low**:

- `PC13 = 1` → LED OFF
- `PC13 = 0` → LED ON

The firmware starts with the LED OFF and then continuously toggles PC13.

---

## 4. Register Base Addresses and Offsets

The firmware accesses peripheral registers directly using memory-mapped addresses.

### RCC

**RCC base address:** `0x40021000`

| Register | Offset | Absolute Address | Purpose |
|---|---:|---:|---|
| `APB2ENR` | `0x18` | `0x40021018` | Enables the GPIOC peripheral clock |

The firmware sets:

```c
RCC_APB2ENR |= (1UL << 4);
```

Bit 4 enables the **GPIOC clock**.

---

### GPIOC

**GPIOC base address:** `0x40011000`

| Register | Offset | Absolute Address | Purpose |
|---|---:|---:|---|
| `CRH` | `0x04` | `0x40011004` | Configures PC8–PC15 |
| `ODR` | `0x0C` | `0x4001100C` | Controls GPIO output state |

#### PC13 configuration

PC13 uses bits `23:20` of `GPIOC_CRH`.

The firmware clears those bits and writes `0x2`:

```c
GPIOC_CRH &= ~(0xFUL << 20);
GPIOC_CRH |= (0x2UL << 20);
```

This configures PC13 as a **2 MHz push-pull output**.

#### LED control

PC13 corresponds to:

```c
#define LED_PIN (1UL << 13)
```

The LED is initially turned OFF:

```c
GPIOC_ODR |= LED_PIN;
```

Then it is toggled using:

```c
GPIOC_ODR ^= LED_PIN;
```

---

### Cortex-M3 SysTick

**SysTick register base:** `0xE000E010`

| Register | Offset | Absolute Address | Purpose |
|---|---:|---:|---|
| `SYSTICK_CTRL` | `0x00` | `0xE000E010` | Enables SysTick and selects processor clock |
| `SYSTICK_LOAD` | `0x04` | `0xE000E014` | Reload/count value |
| `SYSTICK_VAL` | `0x08` | `0xE000E018` | Current counter value |

The delay function uses:

```c
SYSTICK_LOAD = 3999999UL;
SYSTICK_VAL = 0UL;
SYSTICK_CTRL = (1UL << 2) | (1UL << 0);
```

With the STM32's reset/default HSI clock of approximately 8 MHz, 4,000,000 SysTick clock cycles correspond to approximately **0.5 seconds**.

The firmware waits for the SysTick COUNTFLAG:

```c
while ((SYSTICK_CTRL & (1UL << 16)) == 0U)
{
}
```

---

## 5. Boot Flow: Reset to `main()`

The firmware does not use a vendor-provided startup file. The startup sequence is implemented in `src/startup.s`.

### Step 1 — Reset

After reset, the Cortex-M3 reads the first two words of the interrupt vector table:

1. Initial stack pointer → `_estack`
2. Reset handler address → `Reset_Handler`

The vector table is placed at the beginning of FLASH by the linker script.

### Step 2 — `Reset_Handler`

Execution enters:

```asm
Reset_Handler:
```

### Step 3 — Copy `.data` from FLASH to RAM

The linker provides:

- `_sidata` — source address in FLASH
- `_sdata` — beginning of `.data` in RAM
- `_edata` — end of `.data`

The startup code copies initialized global/static data from FLASH into RAM.

### Step 4 — Clear `.bss`

The linker provides:

- `_sbss` — beginning of `.bss`
- `_ebss` — end of `.bss`

The startup code writes zero to the entire `.bss` region.

### Step 5 — Call `main()`

After initialization:

```asm
bl main
```

transfers control to the C program in `src/main.c`.

### Step 6 — Main program

`main()`:

1. Enables the GPIOC clock.
2. Configures PC13 as a push-pull output.
3. Sets PC13 high so the active-low LED starts OFF.
4. Enters an infinite loop.
5. Toggles PC13.
6. Waits approximately 500 ms.
7. Repeats forever.

---

## 6. Memory Map

The project linker script defines:

```text
FLASH: 0x08000000 - 0x0800FFFF   (64 KB)
RAM:   0x20000000 - 0x20004FFF   (20 KB)
```

The linker places:

- `.isr_vector` in FLASH
- `.text` and `.rodata` in FLASH
- `.data` in RAM, with its initial values stored in FLASH
- `.bss` in RAM

---

## 7. Source Files

```text
Task0/
├── Makefile
├── linker.ld
├── diagram.json
├── wokwi.toml
├── task0.elf
├── task0.bin
└── src/
    ├── main.c
    ├── startup.s
    ├── main.o
    └── startup.o
```

### File purpose

**`src/main.c`**  
Contains all application logic, direct register definitions, GPIO setup, SysTick delay, and LED blinking.

**`src/startup.s`**  
Contains the interrupt vector table, reset handler, `.data` initialization, `.bss` clearing, and transfer to `main()`.

**`linker.ld`**  
Defines FLASH/RAM locations, section placement, and the reset entry point.

**`Makefile`**  
Builds the ELF and BIN firmware images and provides `clean` and `flash` targets.

**`diagram.json`**  
Defines the Wokwi STM32 Blue Pill simulation. No external connections are required for the onboard LED test.

**`wokwi.toml`**  
Tells Wokwi to load `task0.bin` and `task0.elf`.

---

## 8. Build Commands

From the `Task0` directory:

### Build

```bash
make
```

This produces:

```text
task0.elf
task0.bin
```

### Clean

```bash
make clean
```

### Flash

For physical hardware with a compatible ST-Link/OpenOCD setup:

```bash
make flash
```

The Makefile uses:

```text
interface/stlink.cfg
target/stm32f1x.cfg
```

For Wokwi simulation, flashing through OpenOCD is not required; Wokwi loads the firmware specified in `wokwi.toml`.

---

## 9. Expected Result

When the firmware runs, the STM32 Blue Pill's onboard **PC13 LED continuously blinks**.

Approximate behavior:

```text
LED ON  → wait ~500 ms
LED OFF → wait ~500 ms
LED ON  → wait ~500 ms
...
```

The LED is controlled entirely through direct GPIO register access.

---

## 10. Design Constraints

This Task 0 implementation intentionally uses:

- Bare-metal C
- ARM Cortex-M3 assembly startup code
- Direct memory-mapped register access
- Custom linker script
- No HAL APIs
- No LL APIs
- No vendor CMSIS startup/system files
- No interrupts required for the LED blink

This keeps the project close to the hardware and demonstrates the complete reset-to-`main()` boot path.
