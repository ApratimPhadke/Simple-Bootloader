# Simple Bootloader on STM32

A bare-metal custom bootloader for the **STM32F411CEUx (BlackPill)** microcontroller, written in C without any HAL or middleware. On startup, it checks a GPIO pin to decide whether to enter **firmware update mode** or **jump directly to the user application**.

---

## Hardware Requirements

| Component | Details |
|---|---|
| Microcontroller | STM32F411CEUx (BlackPill) |
| Programmer / Debugger | ST-LINK V2 |
| IDE | STM32CubeIDE |

### Wiring

- Connect **SWDIO**, **SWDCLK**, **GND**, and **3.3V** from ST-LINK V2 to the corresponding pins on the BlackPill.
- **PA0** — used as the mode-select button input.
- **PC13** — onboard LED on the BlackPill (active low).

---

## Memory Map

```
0x08000000  ──────────────────────────
            │   Bootloader (.text)    │  ← flashed here (sector 0)
0x08004000  ──────────────────────────
            │   User Application      │  ← your app starts here
            ──────────────────────────
```

The bootloader occupies the start of Flash at `0x08000000`. The user application must be compiled and linked to start at `0x08004000`. The bootloader reads the application's Stack Pointer and Reset Handler from this address at runtime.

---

## How It Works

### Boot Decision (PA0 pin check)

On every power-up or reset, the bootloader:

1. Initializes GPIOA (PA0 as input) and GPIOC (PC13 as output).
2. Reads **PA0**:
   - **PA0 = HIGH (button pressed)** → Enter **Update Mode** — blinks LED 3 times and waits for new firmware.
   - **PA0 = LOW (button not pressed)** → **Jump to Application** — blinks LED once and transfers execution to `0x08004000`.

---

## Code Walkthrough

### 1. Register Definitions

```c
#define APP_ADDRESS     0x08004000
#define RCC_AHB1ENR     (*(volatile uint32_t*)0x40023830)
#define GPIOA_MODER     (*(volatile uint32_t*)0x40020000)
#define GPIOA_IDR       (*(volatile uint32_t*)0x40020010)
#define GPIOC_MODER     (*(volatile uint32_t*)0x40020800)
#define GPIOC_ODR       (*(volatile uint32_t*)0x40020814)
```

All peripheral registers are accessed by directly casting their physical addresses to `volatile uint32_t*` pointers. This bypasses any HAL layer entirely. The addresses come from the STM32F411 Reference Manual:

- `RCC_AHB1ENR` — enables clocks for GPIOA and GPIOC peripherals.
- `GPIOA_MODER` / `GPIOC_MODER` — configure each pin as input or output.
- `GPIOA_IDR` — read the current logic level on GPIOA pins.
- `GPIOC_ODR` — write to GPIOC pins to turn the LED on/off.

---

### 2. `init_gpio()`

```c
void init_gpio() {
    RCC_AHB1ENR |= (1U << 0);          // Enable GPIOA clock (bit 0)
    RCC_AHB1ENR |= (1U << 2);          // Enable GPIOC clock (bit 2)
    GPIOA_MODER &= ~(3U << (0 * 2));   // PA0 = Input mode (clear bits [1:0])
    GPIOC_MODER &= ~(3U << (13 * 2));  // PC13 = clear bits first
    GPIOC_MODER |=  (1U << (13 * 2));  // PC13 = Output mode (set bit 26)
}
```

Before any GPIO register can be touched, its peripheral clock must be enabled via `RCC_AHB1ENR`. Without this, reads/writes to GPIO registers have no effect.

Each pin in `MODER` uses 2 bits:
- `00` = Input
- `01` = Output

The code clears both bits first (`&= ~(3U << ...)`) then sets the desired mode — a safe read-modify-write pattern that avoids corrupting other pins.

---

### 3. `led_blink(int n)`

```c
void led_blink(int n) {
    for (int i = 0; i < n; i++) {
        GPIOC_ODR &= ~(1U << 13);           // LED ON  (active low — pulls pin LOW)
        for (volatile int d = 0; d < 200000; d++);  // software delay
        GPIOC_ODR |=  (1U << 13);           // LED OFF (pull pin HIGH)
        for (volatile int d = 0; d < 200000; d++);  // software delay
    }
}
```

The BlackPill's onboard LED on PC13 is **active low** — it turns ON when the pin is driven LOW. The delay loops use `volatile` to prevent the compiler from optimizing them away, giving a visible blink duration at ~100MHz core clock.

Blink count encodes the boot state:
- **3 blinks** = entered Update Mode
- **1 blink** = jumping to application

---

### 4. `jump_to_app()`

```c
void jump_to_app() {
    uint32_t app_sp    = *(volatile uint32_t*) APP_ADDRESS;
    uint32_t app_reset = *(volatile uint32_t*)(APP_ADDRESS + 4);
    __asm volatile("msr msp, %0" :: "r"(app_sp));
    ((void (*)(void))app_reset)();
}
```

This is the core of any bootloader. The ARM Cortex-M vector table always starts with:
- **Word 0 (offset 0x00):** Initial Stack Pointer value
- **Word 1 (offset 0x04):** Address of the Reset Handler function

The bootloader reads these two values directly from the application's start address in Flash, then:
1. **Loads the application's stack pointer** into the MSP (Main Stack Pointer) register using inline assembly (`msr msp, r0`).
2. **Calls the application's Reset Handler** as a function pointer, which hands over full execution to the user app.

After this call, the bootloader is no longer in control — the application runs as if it booted directly.

---

### 5. `main()`

```c
int main() {
    init_gpio();
    if (GPIOA_IDR & (1U << 0)) {
        led_blink(3);
        for (volatile int d = 0; d < 500000; d++);  // stay in update mode
    } else {
        led_blink(1);
        jump_to_app();
    }
    return 0;
}
```

The logic is deliberately simple:
- Read bit 0 of `GPIOA_IDR` to check PA0's state.
- If HIGH → update mode (3 blinks + wait).
- If LOW → jump to app (1 blink + jump).

---

## Building and Flashing

1. Open **STM32CubeIDE** and import the project.
2. Set the linker script so the bootloader is placed at `0x08000000`.
3. Build with **Project → Build All**.
4. Connect the ST-LINK V2 to the BlackPill.
5. Flash with **Run → Debug** or **Run → Run**.
6. Confirm in the console: `Download verified successfully`.

---

## Writing a Compatible User Application

Your user application must be linked to start at `0x08004000`. In STM32CubeIDE, edit the linker script (`.ld` file):

```ld
MEMORY
{
  FLASH (rx) : ORIGIN = 0x08004000, LENGTH = 240K
  RAM  (xrw) : ORIGIN = 0x20000000, LENGTH = 128K
}
```

Also update the **Vector Table Offset Register** at the start of your application's `main()`:

```c
SCB->VTOR = 0x08004000;
```

This tells the Cortex-M4 where to find the interrupt vector table for your application.

---

## Project Structure

```
Bootloader/
├── Src/
│   └── main.c          ← bootloader source code
├── Startup/
│   └── startup_stm32f411ceux.s
├── Bootloader.ld        ← linker script (Flash origin: 0x08000000)
└── README.md
```

---

## Known Limitations

- Update mode currently only waits — actual UART/USB firmware receive logic is not yet implemented.
- No validity check on the application (e.g. checking if `APP_ADDRESS` contains a valid stack pointer before jumping).
- Software delay in `led_blink()` is not calibrated precisely; accuracy depends on clock configuration.

---

## References

- [STM32F411 Reference Manual (RM0383)](https://www.st.com/resource/en/reference_manual/rm0383-stm32f411xce-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)
- [ARM Cortex-M4 Technical Reference Manual](https://developer.arm.com/documentation/100166/latest)
- [STM32CubeIDE User Guide](https://www.st.com/en/development-tools/stm32cubeide.html)
