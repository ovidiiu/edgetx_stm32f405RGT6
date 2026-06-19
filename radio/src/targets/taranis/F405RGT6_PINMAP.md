# STM32F405RGT6 (LQFP64) capsule — EdgeTX pin map

Adaptation of the **TX12 / Zorro / Boxer** (taranis, STM32F407 LQFP100) target to a
custom **STM32F405RGT6 (LQFP64)** capsule.

Scope of this document: **minimal bring-up** — enough to boot, drive the LCD/UI,
read sticks, mount the SD card, talk to one (internal) module, and enumerate over
USB. A rotary encoder, 5 switches and analog trims are wired; external module,
trainer, telemetry and haptic are intentionally deferred (pins reserved below).

---

## 1. Why this is mostly a *pin* job, not a *chip* job

The F405RGT6 and the F407Vx used by the source target are the **same silicon die**.
The EdgeTX build already produces correct firmware for this die when:

| Build setting | Value | Source |
|---|---|---|
| `CPU_TYPE_FULL` | `STM32F407xG` | covers F405/407/415/417, 1 MB flash |
| CMSIS device define | `-DSTM32F40_41xxx -DSTM32F407xx -DSTM32F407xG` | set automatically from `CPU_TYPE_FULL` |
| Linker | `stm32f40x`, `TARGET_FLASH_SIZE 1M` | F405RGT6 = 1 MB flash / 192 KB RAM ✓ |
| Vectors / system | `vectors_stm32f407xx.c`, `system_stm32f4xx.c` | identical |
| **HSE_VALUE** | **12000000** | `targets/taranis/CMakeLists.txt:16` |

> ⚠️ **Hardware constraint:** the clock tree and the 48 MHz USB clock assume a
> **12 MHz HSE crystal**. The capsule must use a 12 MHz crystal on PH0/PH1, or
> `HSE_VALUE` (and the PLL config) must be changed.

So **no new CPU token, CMSIS header, linker, or startup file is needed.** Only the
GPIO/peripheral assignments change, because the LQFP64 package does not bond out
the pins the LQFP100 target relies on.

## 2. The package limitation

| Port | LQFP100 (source) | LQFP64 (RGT6) |
|------|------------------|---------------|
| A | PA0–PA15 | PA0–PA15 ✅ |
| B | PB0–PB15 | PB0–PB15 ✅ |
| C | PC0–PC15 | PC0–PC15 ✅ |
| D | PD0–PD15 | **PD2 only** ⚠️ |
| E | PE0–PE15 | **none** ❌ |
| F / G | present | **none** ❌ |

Total usable I/O drops from **82 → 51**. Anything the source maps to port E, port
F/G, or port D other than PD2 must be reassigned to A/B/C/PD2 — and there simply
aren't enough pins for the full radio, which is why this bring-up is minimal.

## 3. I/O budget (51 pins)

```
51 total
 -4  HSE (PH0/PH1) + LSE 32k (PC14/PC15)
 -8  sticks x4, pots x2, VBAT, audio DAC   (PA0-4,6 / PB0 / PC0)
 -3  USB                                   (PA9/11/12)
 -2  SWD debug                             (PA13/14)
 -6  SD card SDIO 4-bit                    (PC8-12, PD2)
 -4  internal module                       (PB6/7, PC4, PB1)
 -5  LCD hardware SPI3                      (PB3/5/12/13/14)
 -0  backlight hardwired to VCC, no power latch (no pins)
 ~  remainder -> keys / switches / rotary
```

## 4. Pin assignment (minimal bring-up)

### Analog — ADC1 (unchanged from source; already all on A/B/C)
| Func | Pin | Channel |
|---|---|---|
| Stick RV | PA0 | ADC1_IN0 |
| Stick RH | PA1 | ADC1_IN1 |
| Stick LV | PA2 | ADC1_IN2 |
| Stick LH | PA3 | ADC1_IN3 |
| Pot P1 | PB0 | ADC1_IN8 |
| Trim T1 (pot `P2`) | PA6 | ADC1_IN6 |
| Trim T2 (pot `P3`) | PC3 | ADC1_IN13 |
| Trim T3 (pot `P4`) | PC1 | ADC1_IN11 |
| Trim T4 (pot `P5`) | PC2 | ADC1_IN12 |
| VBAT sense | PC0 | ADC1_IN10 |
| Audio out | PA4 | DAC1_OUT1 (DMA1_Str5, TIM6) |

> The 4 analog trims are wired as ordinary analog pots (`P2`–`P5`, labelled
> T1–T4). EdgeTX has no native "analog trim" concept — use them as mixer input
> sources to act as trims. One ADC1 DMA scan covers all 9 axes + VBAT.

### System
| Func | Pin | AF/periph |
|---|---|---|
| USB VBUS | PA9 | OTG_FS |
| USB DM | PA11 | OTG_FS AF10 |
| USB DP | PA12 | OTG_FS AF10 |
| SWDIO | PA13 | debug |
| SWCLK | PA14 | debug |
| HSE | PH0/PH1 | 12 MHz xtal |
| LSE | PC14/PC15 | 32.768 kHz xtal |

### Power
No soft-power circuit on the WeAct dev board (powered from USB/VIN via a
hardware switch). `PWR_SWITCH_GPIO`/`PWR_ON_GPIO` are **undefined**, so PC1/PC2
are free for analog trims T3/T4. With no power macros, `pwrPressed()` returns
true and the (non-`PWR_BUTTON_PRESS`) `pwrCheck()` keeps the radio powered on.

### Internal module — USART1 (AF7)
| Func | Pin |
|---|---|
| INT_TX | PB6 (USART1_TX) |
| INT_RX | PB7 (USART1_RX) |
| INT_PWR | PC4 (out) |
| INT_BOOTCMD | PB1 (out) |

DMA: TX = DMA2_Stream7_Ch4, RX = DMA2_Stream2_Ch4 (as source).

### SD card — onboard microSD via SDIO 4-bit (AF12)
| Func | Pin |
|---|---|
| SDIO_D0 | PC8 |
| SDIO_D1 | PC9 |
| SDIO_D2 | PC10 |
| SDIO_D3 | PC11 |
| SDIO_CK | PC12 |
| SDIO_CMD | PD2 |
| SD_PRESENT | *none* — card assumed always present |

Pins are the fixed STM32F4 SDIO assignment (hardcoded AF12 in the SDIO driver),
so PC8-12 + PD2 are consumed here. DMA: DMA1_Stream3. `STORAGE_USE_SDIO`.

### LCD — hardware SPI3 (AF6)
Relocated off port C onto PB.03/PB.05 so PC.10-12 are free for the SDIO data
lines above; control lines on PB.12-14 (freed from the old SPI2 SD wiring).
| Func | Pin |
|---|---|
| LCD_CLK | PB3 (SPI3_SCK) |
| LCD_MOSI | PB5 (SPI3_MOSI) |
| LCD_A0 (D/C) | PB12 (GPIO) |
| LCD_NCS | PB13 (GPIO) |
| LCD_RST | PB14 (GPIO) |
| Backlight | *no pin* — hardwired to VCC (always on); see below |

DMA: DMA1_Stream7. Driver: `lcd_driver_spi.cpp` (128×64 mono). Match the
controller to your physical display.

> **Backlight:** the panel backlight is tied directly to VCC, so there is no
> firmware brightness control. `BACKLIGHT_GPIO` is left undefined, which makes
> `backlight_driver.cpp` compile to no-op stubs and frees **PA.10 for switch
> SC**.

### I²C1 (EEPROM) — AF4
| Func | Pin |
|---|---|
| I2C1_SCL | PB8 |
| I2C1_SDA | PB9 |
| EEPROM_WP | PC7 |

> Populate a 24Cxx EEPROM on PB8/PB9, or move settings storage to SD.

### Trainer — disabled
No trainer port in this build: its former pins PC.08/PC.09 are now the SDIO
microSD data lines (SDIO_D0/D1). Re-adding trainer means finding free TIM
channels elsewhere — there is no spare pin pair in the minimal map.

### Menu navigation — rotary encoder + keys
Primary navigation is a **rotary encoder** (NAVIGATION_X7_RM maps page change to
rotation, so the knob both scrolls fields and changes pages). Push = ENTER.
| Function | Pin | Notes |
|---|---|---|
| Encoder A | PB10 | EXTI10, internal pull-up |
| Encoder B | PB11 | EXTI11, internal pull-up |
| ENTER (encoder push) | PA5 | KEY_ENTER, active-low |
| EXIT | PC13 | back/cancel |
| MDL | PB4 | model select menu |
| SYS | PA7 | radio settings menu |
| TELE | PB15 | telemetry view (optional) |

Minimum to navigate everything: encoder + EXIT + MDL + SYS. PAGEUP/PAGEDN keys
removed (their pins are now the encoder); paging is done by the rotary.

### Switches (5× 2-position, GPIO, active-low pull-up)
| Switch | Pin | Notes |
|---|---|---|
| SA | PC5 | |
| SB | PA15 | |
| SC | PA10 | freed from backlight (backlight hardwired to VCC) |
| SD | PA8 | |
| SE | PC7 | shares pin with EEPROM_WP — OK while no I²C EEPROM is fitted |

## 5. Pins reserved for later expansion

The minimal map is now nearly full — the SDIO microSD (PC8-12 + PD2) and the
relocated LCD (PB3/PB5/PB12-14) consumed most of the pins this table previously
offered. What is realistically left:

| Future function | Suggested pins | Constraint |
|---|---|---|
| External module | PC6 (TIM8_CH1 / USART6_TX) | RX/PWR pins must be stolen from a switch/key; PB3/PB5 are no longer free (LCD) |
| Haptic | PC6 (declared, inert) | currently the only spare; conflicts with ext-module TX |
| Telemetry (S.PORT) | **see warning below** | |

Trainer, a second rotary, and an I²C EEPROM no longer have free pins in this
build (their former candidates PC8/PC9, PB10/PB11, PB3/PB5 are all in use).

Still free after the above: only **PB2** (BOOT1 — input-only, avoid driving).

## 6. Known hard constraints (read before adding peripherals)

1. **Telemetry cannot stay on USART2.** EdgeTX telemetry is wired to `USART2`,
   whose only pin options are PA2/PA3 (used by sticks) or PD5/PD6 (gone on LQFP64).
   To add telemetry you must move it to another USART (e.g. **USART3** on
   PB10/PB11 or PC10/PC11, or **UART5** TX=PC12/RX=PD2) **in the driver/header**,
   not just by renaming a pin. This is a code change, not a pin swap.
2. **USB needs the 12 MHz HSE crystal** (see §1).
3. **PB2 = BOOT1** and **PA13/PA14 = SWD** — keep their boot/debug roles in mind.
4. Sticks/pots intentionally kept on their source pins so the ADC ladder, sample
   times and DMA stream (DMA2_Stream4) are unchanged.

## 7. Files to change

1. `radio/src/boards/hw_defs/f405rgt6.json` — analog inputs, switches, keys, trims
   (created alongside this doc).
2. `radio/src/targets/taranis/CMakeLists.txt` — add a `PCBREV STREQUAL F405RGT6`
   branch: `FLAVOUR f405rgt6`, `CPU_TYPE_FULL STM32F407xG`, `ROTARY_ENCODER NO`,
   minimal module config.
3. `radio/src/targets/taranis/hal.h` — add `defined(RADIO_F405RGT6)` cases (or a
   single guarded block) for every macro the source maps to ports D/E/F/G:
   PWR_SWITCH/PWR_ON, INTMODULE_PWR/BOOTCMD, LCD_*, SD_PRESENT, status LEDs,
   rotary, telemetry, ext-module — using the pins in §4.

Build (from `radio/`):
```
cmake -DPCB=X7 -DPCBREV=F405RGT6 ...
```

## 8. Status / next steps

- [x] Confirm chip = F407xG build path, 1 MB, 12 MHz HSE
- [x] Master pin map (this doc) + minimal `f405rgt6.json`
- [x] Add CMake `PCBREV F405RGT6` branch
- [x] Add `RADIO_F405RGT6` overrides in `hal.h`
- [x] Register flavour in `tools/build-common.sh` + CI workflow `build_f405rgt6.yml`
- [ ] Green CI build (fix any remaining D/E/F/G references the preprocessor pulls in)
- [ ] Flash + bring up LCD, sticks, SD, USB

### Build it
- **Cloud (no local toolchain):** push to GitHub → the **Build F405RGT6 capsule**
  Action compiles in the EdgeTX container and uploads `*.bin` as an artifact.
- **Local (if you ever get a toolchain):**
  `cmake --fresh -S radio -B build -DPCB=X7 -DPCBREV=F405RGT6 -DCMAKE_BUILD_TYPE=Release && make -C build -j firmware`

### Still inert / non-functional in this build (documented, not wired)
- **Telemetry** stays on USART2 (PD4/5/6) — compiles but won't work until moved
  to another USART in the driver.
- **External module / heartbeat** disabled (`HARDWARE_EXTERNAL_MODULE NO`).
- **Status LEDs** disabled (`STATUS_LEDS NO`); **haptic** declared but inert (PC6).
- **Backlight** has no firmware control — hardwired to VCC; PA10 reused for switch SC.

Wired and functional: rotary encoder (PB10/PB11 + push PA5), 5 switches, analog
trims, onboard microSD over SDIO.
