# STM32F405RGT6 (LQFP64) capsule — EdgeTX pin map

Adaptation of the **TX12 / Zorro / Boxer** (taranis, STM32F407 LQFP100) target to a
custom **STM32F405RGT6 (LQFP64)** capsule.

Scope of this document: **minimal bring-up** — enough to boot, drive the LCD/UI,
read sticks, mount the SD card, talk to one (internal) module, and enumerate over
USB. External module, trainer, telemetry, haptic, rotary encoder, switches and
trims are intentionally deferred (pins reserved below).

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
 -4  SD card SPI2                          (PB12-15)
 -4  internal module                       (PB6/7, PC4, PB1)
 -5  LCD soft-SPI                          (PC10/11/12, PD2, PC3)
 -2  backlight + power latch
 ~  remainder -> keys / future peripherals
```

## 4. Pin assignment (minimal bring-up)

### Analog — ADC1 (unchanged from source; already all on A/B/C)
| Func | Pin | Channel |
|---|---|---|
| Stick RV | PA0 | ADC1_IN0 |
| Stick RH | PA1 | ADC1_IN1 |
| Stick LV | PA2 | ADC1_IN2 |
| Stick LH | PA3 | ADC1_IN3 |
| Pot P2 | PA6 | ADC1_IN6 |
| Pot P1 | PB0 | ADC1_IN8 |
| VBAT sense | PC0 | ADC1_IN10 |
| Audio out | PA4 | DAC1_OUT1 (DMA1_Str5, TIM6) |

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
| Func | Pin | Dir |
|---|---|---|
| PWR_SWITCH (read button) | PC1 | in |
| PWR_ON (latch) | PC2 | out |

### Internal module — USART1 (AF7)
| Func | Pin |
|---|---|
| INT_TX | PB6 (USART1_TX) |
| INT_RX | PB7 (USART1_RX) |
| INT_PWR | PC4 (out) |
| INT_BOOTCMD | PB1 (out) |

DMA: TX = DMA2_Stream7_Ch4, RX = DMA2_Stream2_Ch4 (as source).

### SD card — SPI2 (AF5)
| Func | Pin |
|---|---|
| SD_SCK | PB13 |
| SD_MISO | PB14 |
| SD_MOSI | PB15 |
| SD_CS | PB12 (soft) |
| SD_PRESENT | *none* — card assumed always present |

DMA: DMA1_Stream3 (RX) / Stream4 (TX), Ch0.

### LCD — hardware SPI3 (AF6), same as PCBX7 except RST
| Func | Pin |
|---|---|
| LCD_CLK | PC10 (SPI3_SCK) |
| LCD_MOSI | PC12 (SPI3_MOSI) |
| LCD_A0 (D/C) | PC11 (GPIO) |
| LCD_NCS | PA15 (GPIO) |
| LCD_RST | PC3 (GPIO — moved off PD12) |
| Backlight | PA10 (TIM1_CH3 AF1, BDTR/MOE set) |

DMA: DMA1_Stream7. Driver: `lcd_driver_spi.cpp` (128×64 mono). Match the
controller to your physical display.

### I²C1 (EEPROM) — AF4
| Func | Pin |
|---|---|
| I2C1_SCL | PB8 |
| I2C1_SDA | PB9 |
| EEPROM_WP | PC7 |

> Populate a 24Cxx EEPROM on PB8/PB9, or move settings storage to SD.

### Trainer — TIM3 (AF2), defaults already on safe pins
| Func | Pin |
|---|---|
| TRAINER_IN | PC8 (TIM3_CH3) |
| TRAINER_OUT | PC9 (TIM3_CH4) |
| TRAINER_DETECT | PA8 |

### Navigation keys (GPIO, active-low)
| Key | Pin |
|---|---|
| EXIT | PC13 |
| ENTER | PA5 |
| PAGEUP | PB10 |
| PAGEDN | PB11 |
| MDL | PB3 |
| TELE | PB4 |
| SYS | PB5 |
| SA (test switch) | PC5 |

## 5. Pins reserved for later expansion

| Future function | Suggested pins | Constraint |
|---|---|---|
| External module | PC6 (TIM8_CH1 / USART6_TX), PC7 (USART6_RX), PWR=PB5 | PPM/PXX needs TIM8_CH1 → PC6 |
| Trainer | PC8 (TIM3_CH3), PC9 (TIM3_CH4), detect=PB9 | |
| Haptic | PB3 (TIM2_CH2 AF1) | |
| Rotary encoder | PB10 + PB11 (EXTI) | |
| I²C EEPROM / IMU | PB10/PB11 (I²C2 AF4) | conflicts with rotary — pick one |
| Telemetry (S.PORT) | **see warning below** | |

Still free after the above: PB2 (BOOT1 — input-only, avoid driving), PB5.

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
- **Status LEDs, haptic, rotary** disabled.
