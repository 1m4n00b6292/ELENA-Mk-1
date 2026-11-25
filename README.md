# ELENA Mk.1

**ELENA Mk.1** is a Raspberry Pi HAT built around an **STM32H562RGT6** (Cortex-M33, LQFP64). It sits between a
Raspberry Pi (developed against a Pi 5) and the robot's low-level hardware:

- The **Pi talks to the MCU over SPI**: fixed 64-byte frames with CRC, the MCU as SPI slave, plus a data-ready line.
- A **UART link to the Pi (or a USB-UART adapter)** gives a text console for bring-up, quick debugging and manual control.
- The MCU drives **four UNIDRIVE wheel drivers over CAN** (250 kbit/s) and samples a **TDK ICM-42688-P IMU** at 1 kHz.
- The board takes a **24 V-class supply** and generates 5 V (also fed to the Pi) and 3.3 V on board.

The firmware runs on **FreeRTOS** and is built in layers (L1 BSP → L2 RTOS + bus drivers → L3 device drivers →
L4 Pi link + parameters → L5 control). The current version is **0.4.0 (L4)**: the Pi link, telemetry and the
parameter store work; motion commands over SPI are refused until the L5 control layer lands.

---

## Contents

- [Hardware](#hardware)
- [Firmware architecture](#firmware-architecture)
- [SPI link protocol (Pi ⇄ MCU)](#spi-link-protocol-pi--mcu)
- [UART console](#uart-console)
- [CAN: UNIDRIVE wheel drivers](#can-unidrive-wheel-drivers)
- [Parameters](#parameters)
- [Status LEDs and fault codes](#status-leds-and-fault-codes)
- [Build and flash](#build-and-flash)
- [Host unit tests](#host-unit-tests)
- [Pi-side tool: `elena_link.py`](#pi-side-tool-elena_linkpy)
- [Bench tests](#bench-tests)
- [Repository layout](#repository-layout)
- [Known issues and notes](#known-issues-and-notes)

---

## Hardware

Schematic: [`ELENA_Mk_1.pdf`](ELENA_Mk_1.pdf) (KiCad 10, five sheets: top level, Protection, DC_Power,
MCU_STM32H562RGT6, CAN_TRANSCEIVER).

### Block overview

```
 VIN (J1) ──► Protection ──► +24V ──► LMR51450 buck ──► +5V ──┬──► Pi 5V (J2 pin 2)
              LM74502 + 2x CSD18510Q5B                         ├──► TJA1051T CAN transceiver
              TVS3301, SMBJ5371B                               └──► AP2112K-3.3 ──► +3V3 (MCU)

 Raspberry Pi header (J2) ◄── SPI0 + DRDY ──► STM32H562RGT6 ◄── SPI3 + I2C1 ──► J4 sensor header (ICM-42688-P)
                          ◄── UART ───────►               ──── FDCAN1 ─────► TJA1051T ──► J5 CANH/CANL
                                                          ──── SWD ────────► J3
```

### Power

| Stage | Part | Notes |
| --- | --- | --- |
| Input | J1 (2-pin) | VIN with LED indicator |
| Reverse / over-voltage protection | LM74502H controller driving back-to-back CSD18510Q5B MOSFETs | OV threshold set by R3/R4 (100 k / 3.5 k); TVS3301 (37 V abs max) and SMBJ5371B on the input |
| 24 V → 5 V | LMR51450 synchronous buck, 4.7 µH | Output +5V, test point TP2 |
| 5 V → 3.3 V | AP2112K-3.3 LDO | Output +3V3, test point TP3 |

The +5V rail is connected to the Pi header's 5V pin, so the HAT can power the Pi.

### MCU and clocks

- STM32H562RGT6, 8 MHz crystal (Y1) on PH0/PH1, PLL to **168 MHz** SYSCLK.
- 1 MB flash, 640 KB RAM (per the linker script).
- BOOT0 strapped through R8 (10 k), 100 nF on NRST.

### Pinout highlights

| Function | MCU pin | Connects to | Firmware |
| --- | --- | --- | --- |
| SPI1 NSS | PA4 | Pi GPIO8 / CE0 (header pin 24) | SPI1 slave, hardware NSS input |
| SPI1 SCK | PA5 | Pi GPIO11 / SCLK (pin 23) | |
| SPI1 MISO | PA6 | Pi GPIO9 / MISO (pin 21) | |
| SPI1 MOSI | PA7 | Pi GPIO10 / MOSI (pin 19) | |
| SPI1 DRDY | PC4 | Pi GPIO7 (pin 26) | GPIO output, "response ready" |
| USART1 TX | PB14 | Pi GPIO15 / RXD (pin 10) | 115200 8N1 console |
| USART1 RX | PB15 | Pi GPIO14 / TXD (pin 8) | |
| FDCAN1 RX / TX | PA11 / PA12 | TJA1051T (U6) → J5 | Classic CAN, 250 kbit/s |
| SPI3 SCK / MISO / MOSI | PC10 / PC11 / PC12 | J4 | SPI master, 6 MHz, mode 0 |
| SPI3 CS1 / CS2 / CS3 | PD2 / PB3 / PB4 | J4 | GPIO chip selects (CS1 = IMU) |
| I2C1 SCL / SDA | PB6 / PB7 | J4 | Configured, not used yet |
| SWDIO / SWCLK | PA13 / PA14 | J3 | Debug |
| LED_1 … LED_4 | PC3, PC2, PC1, PC0 | D6, D5, D4, D3 (510 Ω to GND) | See [LEDs](#status-leds-and-fault-codes) |

### Connectors

| Ref | Type | Pins |
| --- | --- | --- |
| J1 | 1x2 | Power input |
| J2 | Raspberry Pi 40-pin | 5V, GND, SPI0 (GPIO8–11), GPIO7 (DRDY), UART (GPIO14/15) |
| J3 | 1x3 SWD | 1 SWDIO, 2 SWCLK, 3 GND |
| J4 | 2x5 sensor header | 1 SDA, 2 SCL, 3 SPI3_CS3, 4 SPI3_CS2, 5 GND, 6 +5V, 7 SPI3_CS1, 8 SPI3_MOSI, 9 SPI3_MISO, 10 SPI3_SCK |
| J5 | 1x2 CAN | CANH, CANL. 120 Ω termination (R13) selectable with jumper JP1 |

> **J4 supply:** J4 only carries +5V, but the ICM-42688-P is rated 3.6 V max. Power the IMU module from 3.3 V
> (see [Known issues](#known-issues-and-notes)).

### IMU

TDK ICM-42688-P on SPI3 / CS1 (datasheet: [`ds-000347-icm-42688-p-v1.6.pdf`](ds-000347-icm-42688-p-v1.6.pdf)).
Default setup: accel ±4 g, gyro ±500 dps, both at 1 kHz ODR, low-noise mode. J4 has no interrupt line, so the
firmware polls it every 1 ms with a 14-byte burst read (about 20 µs at 6 MHz).

---

## Firmware architecture

STM32CubeMX project (`ELENA_Mk1_FW_V1.ioc`) with CMake, HAL, and a vendored **FreeRTOS V11.1.0** kernel
(port `ARM_CM33_NTZ`, non-secure, no TrustZone, `heap_4`). Application code lives in `App/` and is layered:

| Layer | Folder | What it does |
| --- | --- | --- |
| L1 BSP | `App/bsp/` | Reset cause, CPU fault capture (register dump on UART, kept in backup SRAM across a reset), LEDs, µs time base (TIM2), polled debug UART, IWDG |
| L2 bus drivers | `App/drivers/bus/` | USART1 with DMA both ways, SPI3 master, FDCAN1 with RX filters and a TX ring, CRC-16 on the CRC peripheral plus software CRC-16/CRC-8 |
| L3 device drivers | `App/drivers/dev/` | ICM-42688-P, UNIDRIVE CAN protocol (pure packing/parsing) and node tracking |
| L4 comms | `App/comms/` | SPI frame format (pure code), SPI1 slave link with DRDY, parameter table and flash storage |
| App | `App/app/` | Tasks, self-test, bring-up shell and console commands, LED blink codes, link application glue |
| OS | `App/os/` | `FreeRTOSConfig.h`, RTOS hooks |

Pure code (`spi_proto.c`, `unidrive_proto.c`, `params.c`, `crc_sw.c`) has no HAL or RTOS dependency and is unit
tested on a PC.

### Tasks (firmware 0.4.0)

| Task | Priority | Job |
| --- | --- | --- |
| `link` | 6 | SPI1 slave link to the Pi: validate each frame, answer, re-arm DMA, raise DRDY |
| `imu` | 4 | Poll the ICM-42688-P at 1 kHz, publish the latest sample, re-check WHO_AM_I every 100 ms |
| `canrx` | 3 | Hand each received CAN frame to the UNIDRIVE driver |
| `boot` | 2 | Start bus drivers, load parameters, run the self-test, start the other tasks, then run the console shell |
| `hb` | 1 | Every 20 ms: kick the IWDG, update LED blink codes; toggle the heartbeat LED at 1 Hz |

L5 adds the control task (state machine, kinematics, fault manager).

### Boot self-test

At every boot the firmware checks: software and peripheral CRC-16 and CRC-8 against their check values, UART TX
through DMA, CAN internal loopback into both FIFOs, CAN hardware filter rejection, and the IMU WHO_AM_I (0x47).
The result prints on the console and a failed internal check shows as LED blink code 4-1. Run it again any time
with `selftest`.

### Build-time switches

`App/Inc/config.h`:

| Switch | Meaning |
| --- | --- |
| `CFG_IWDG_ENABLE`, `CFG_IWDG_TIMEOUT_MS` | Independent watchdog on/off, timeout (100 ms) |
| `CFG_IWDG_FREEZE_IN_DEBUG` | Freeze the IWDG while the debugger halts the core |
| `CFG_HOST_WDT_ENABLE`, `CFG_HOST_TIMEOUT_MS` | Raise HOST_TIMEOUT if no valid SPI frame for 500 ms (off for bench work without a Pi) |
| `CFG_CMD_STALE_MS` | Ramp to zero when the newest drive command is older than this |
| `CFG_FAULT_RESET` | After a CPU fault: 1 = reset, 0 = halt and blink LED_3 so a debugger can attach |
| `CFG_TRAP_DIV_BY_ZERO` | Trap integer divide by zero as a UsageFault |
| `CFG_DRV_EXPECTED_MASK` | Which wheel driver nodes must be present (0x0F = all four) |
| `CFG_LED_UART_RX` | LED_2 flashes on console RX (until L5 uses it for ARMED) |
| `CFG_TEST_FAULT`, `CFG_TEST_IWDG` | Bench-only: deliberate BusFault / hang 5 s after boot |

Enable both watchdogs before running on the robot.

---

## SPI link protocol (Pi ⇄ MCU)

The Pi is SPI master (SPI0, CE0, mode 0, 8-bit, MSB first; the Pi tool defaults to 4 MHz). The MCU is the slave on
SPI1 with hardware NSS.

### Transaction flow

1. The MCU has a 64-byte response loaded and both DMAs armed, then raises **DRDY** (PC4 → Pi GPIO7).
2. The Pi asserts CE0 and clocks exactly 64 bytes each way. NSS falling drops DRDY.
3. NSS rising wakes the `link` task: it checks that exactly 64 bytes arrived, validates and handles the command,
   builds the next response, re-arms DMA and raises DRDY again.

The response clocked out during a transfer was prepared after the **previous** transfer, so the answer to a command
arrives in the next transaction. A transfer of any other length is aborted, counted, and answered with ACK result 6
(`SHORT`).

### Frame format (64 bytes, all fields little-endian)

| Bytes | Field | Notes |
| --- | --- | --- |
| 0 | SOF | `0xA5` |
| 1 | VER | `0x01` |
| 2 | TYPE | See below |
| 3 | SEQ | Sequence number |
| 4 | LEN | Payload length, must match the TYPE exactly |
| 5 | FLAGS | Response: bit 0 another param reply waiting, bit 1 faults latched |
| 6..61 | Payload | 56 bytes, zero-padded |
| 62..63 | CRC | CRC-16/CCITT-FALSE (poly 0x1021, init 0xFFFF) over bytes 0..61 |

### Commands (Pi → MCU)

| TYPE | Name | LEN | Payload |
| --- | --- | --- | --- |
| `0x00` | POLL | 0 | |
| `0x01` | DRIVE_TWIST | 8 | `vx`, `vy`, `wz` (i16), `accel_max` (u16, 0 = use parameter) |
| `0x02` | WHEEL_SPEED | 8 | 4 × i16, 0.1 rpm |
| `0x03` | WHEEL_DUTY | 8 | 4 × i16, 0.01 % |
| `0x10` | ARM | 3 | key `0x4D52`, arm (u8) |
| `0x11` | ESTOP | 0 | |
| `0x12` | CLEAR_FAULTS | 4 | mask (u32) |
| `0x20` | SET_PARAM | 6 | id (u16), value (f32) |
| `0x21` | GET_PARAM | 2 | id (u16) |
| `0x22` | SAVE_PARAMS | 2 | key `0x5356` |
| `0x7F` | REBOOT | 3 | key `0x5242`, target (u8) |

SET_PARAM, SAVE_PARAMS and REBOOT are refused while armed. At L4 the robot cannot be armed: ARM, DRIVE_TWIST,
WHEEL_SPEED and WHEEL_DUTY are answered `REFUSED`; ESTOP and CLEAR_FAULTS go straight to the drivers.

### Responses (MCU → Pi)

**TELEMETRY (`0x80`)**, payload offsets:

| Offset | Field | Type / unit |
| --- | --- | --- |
| 0 | ack_seq | u8, SEQ of the command being acknowledged |
| 1 | ack_result | u8: 0 OK, 1 BAD_CRC (also bad SOF/VER), 2 UNKNOWN_TYPE, 3 BAD_LEN, 4 REFUSED, 5 BAD_KEY, 6 SHORT |
| 2 | state | u8: 0 BOOT, 1 DISARMED, 2 ARMED, 3 ESTOP, 4 FAULT |
| 3 | drv_online | u8 bit mask of wheel drivers |
| 4 | faults | u32 bit mask |
| 8 | time_us | u32 |
| 12 | acc[3] | i16 raw counts |
| 18 | gyr[3] | i16 raw counts |
| 24 | imu_temp | i16 raw (°C = raw / 132.48 + 25) |
| 26 | wheel_spd[4] | i16, 0.1 rpm |
| 34 | wheel_pos[4] | i32, unwrapped encoder counts |
| 50 | motor_i[4] | u8, 0.1 A magnitude |
| 54 | vbus | u16, 10 mV |

**PARAM_REPLY (`0x81`)**: id (u16), status (0 OK, 1 UNKNOWN, 2 RANGE, 3 READONLY, 4 ARMED), value (f32).

**Fault bits** set at L4: 3 IMU_FAIL, 4–7 DRV1–4_OFFLINE, 8–11 DRV1–4_FAULT, 12 CAN_BUS_OFF, 13 CAN_ERR_PASSIVE,
18 WDT_RESET, 19 PARAM_INVALID.

### Pi setup

```
# /boot/firmware/config.txt
dtparam=spi=on
dtoverlay=spi0-1cs     # one chip select only, frees GPIO7 for DRDY
```

---

## UART console

USART1 at **115200 8N1**, no flow control, DMA in both directions. Reach it from the Pi's UART (GPIO14/15) or with a
3.3 V USB-UART adapter on PB14 (MCU TX) / PB15 (MCU RX). Set the terminal to treat `\r\n` as one newline.

After boot the console prints a banner (firmware version, SYSCLK, reset cause, watchdog state), the self-test
results and a `>` prompt. Type `help` for the full list.

| Group | Commands |
| --- | --- |
| System | `help`, `stat`, `top`, `selftest`, `faults`, `crc <text>`, `wdtstall`, `reboot` |
| IMU / SPI3 | `imu [reset]`, `imustream [hz]`, `spi <cs 1-3> <hex bytes>` |
| CAN | `cantx <id> [bytes]`, `canburst <n>`, `canmon`, `canping [count] [n]`, `canmode normal\|silent\|loop`, `canrecover` |
| Wheel drivers | `drv [clear]`, `drvspeed <n> <rpm> [s]`, `drvduty <n> <%> [s]`, `drvctl enable\|disable\|estop\|clear [n]`, `drvenc <n>`, `drvparam <n> read <idx> \| write <idx> <value> \| save` |
| Pi link (L4) | `link [clear\|raw]`, `linkmon`, `status` |
| Parameters (L4) | `param [list]`, `param get <id\|name>`, `param set <id\|name> <value>`, `param save`, `param defaults` |

---

## CAN: UNIDRIVE wheel drivers

FDCAN1 in classic CAN 2.0A mode at **250 kbit/s**. Four wheel driver nodes: 1 FL, 2 FR, 3 RL, 4 RR. CAN ID =
function code + node id; multi-byte fields are little-endian, and periodic frames end in a CRC-8/SAE-J1850 over
bytes 0..6.

| Function | Base ID | Direction |
| --- | --- | --- |
| CONTROL (enable, disable, e-stop, clear) | `0x000` | MCU → drivers |
| FAULT_EVENT | `0x080` | drivers → MCU |
| SPEED_CMD (coast, brake, speed, duty) | `0x200` | MCU → driver |
| FEEDBACK | `0x280` | driver → MCU |
| STATUS | `0x300` | driver → MCU |
| PARAM_RESP | `0x580` | driver → MCU |
| PARAM_REQ | `0x600` | MCU → driver |
| BOOT | `0x700` | driver → MCU |

Hardware filters accept only the driver-to-MCU IDs: FAULT_EVENT and FEEDBACK go to FIFO0, STATUS, PARAM_RESP and
BOOT to FIFO1. Both FIFOs drain into one 32-frame RTOS queue; transmit uses a 32-frame software ring so a 100 Hz
burst never blocks the caller. A node counts as online if any frame arrived in the last 300 ms.

---

## Parameters

16 float parameters, shared by SPI (`SET_PARAM` / `GET_PARAM` / `SAVE_PARAMS`) and the console (`param`). Each has a
range and a default. They are stored in the last 8 KB flash sector (bank 2) with a CRC; a missing or bad block loads
the defaults and raises `PARAM_INVALID` until the next successful save. IDs are stable: append only, never renumber.

| ID | Name | Default | Range | Unit |
| --- | --- | --- | --- | --- |
| 0 | table_version | – | read only | |
| 1 | track_width | 400 | 50–2000 | mm |
| 2 | wheel_diam | 150 | 20–1000 | mm |
| 3 | enc_cpr | 8192 | 1–1e6 | counts/rev |
| 4 | max_wheel_rpm | 300 | 1–3000 | rpm |
| 5 | accel_max | 1000 | 10–20000 | mm/s² |
| 6 | cmd_stale | 200 | 20–2000 | ms |
| 7 | host_timeout | 500 | 50–5000 | ms |
| 8 | drv_offline | 30 | 10–1000 | ms |
| 9 | vbus_low | 20 | 0–40 | V |
| 10 | vbus_high | 30 | 0–40 | V |
| 11 | tilt_max | 30 | 1–90 | deg |
| 12 | tilt_time | 200 | 0–5000 | ms |
| 13 | imu_accel_fs | 2 (±4 g) | 0–3 | 0 = 16 g … 3 = 2 g, next boot |
| 14 | imu_gyro_fs | 2 (±500 dps) | 0–4 | 0 = 2000 dps … 4 = 125 dps, next boot |
| 15 | spare15 | 0 | | reserved |

---

## Status LEDs and fault codes

| LED | Pin | Ref | Meaning |
| --- | --- | --- | --- |
| LED_1 | PC3 | D6 | Heartbeat, 1 Hz |
| LED_2 | PC2 | D5 | Flashes on console RX (becomes ARMED at L5) |
| LED_3 | PC1 | D4 | Fault blink codes (below); fast blink = CPU fault or RTOS assert |
| LED_4 | PC0 | D3 | Toggles every 10 valid SPI frames |

LED_3 blink codes: number of long flashes = subsystem, number of short flashes = problem. Codes clear by
themselves when the cause goes away; several faults are shown one after the other.

| Code | Meaning |
| --- | --- |
| 1-1 | Motor driver offline (an expected node not heard for 300 ms) |
| 1-2 | Motor driver reports FAULT or E-STOP |
| 2-1 | CAN bus-off |
| 2-2 | CAN error-passive (frames not ACKed: no other node, cable, termination) |
| 3-1 | IMU not answering |
| 4-1 | Boot self-test failed (stays until a `selftest` passes) |
| 5-1 | Console UART receiver not running for 200 ms |

---

## Build and flash

**Requirements:** STM32CubeMX (STM32CubeH5 package), Arm GNU toolchain (`arm-none-eabi-gcc`), CMake ≥ 3.22,
Ninja, and an ST-Link (or STM32CubeProgrammer). VS Code with the STM32 extension or STM32CubeIDE also work.

1. Open `firmware/ELENA_Mk1_FW_V1.ioc` in STM32CubeMX and **Generate Code** (toolchain: CMake). This creates the
   HAL drivers, startup file, `cmake/stm32cubemx/` and the CMake presets that the top-level `CMakeLists.txt` uses.
   The FreeRTOS kernel is vendored in `Middlewares/Third_Party/FreeRTOS` and is not managed by CubeMX.
2. Build:
   ```sh
   cd firmware
   cmake --preset Debug
   cmake --build --preset Debug
   ```
   Output: `build/Debug/ELENA_Mk1_FW_V1.elf`.
3. Flash over SWD (J3: SWDIO, SWCLK, GND), for example:
   ```sh
   STM32_Programmer_CLI -c port=SWD -w build/Debug/ELENA_Mk1_FW_V1.elf -v -rst
   ```
4. Open the console at 115200 8N1 and check the banner and self-test.

To decode a fault dump: `arm-none-eabi-addr2line -e build/Debug/ELENA_Mk1_FW_V1.elf <PC>`.

---

## Host unit tests

Hardware-independent code is tested on a PC with a normal C compiler (gcc, clang or MSVC; not the ARM toolchain):

```sh
cd firmware
cmake -S tests/host -B build-host
cmake --build build-host
ctest --test-dir build-host --output-on-failure
```

Four tests, all must pass:

| Test | Covers |
| --- | --- |
| `crc` | CRC-16/CCITT-FALSE (check 0x29B1) and CRC-8/SAE-J1850 (check 0x4B) |
| `unidrive` | UNIDRIVE CAN frame packing and parsing |
| `spi_proto` | SPI frame build/parse, rebuilds the spec's DRIVE_TWIST and TELEMETRY examples byte for byte |
| `params` | Parameter table, range checks, flash image build/parse |

---

## Pi-side tool: `elena_link.py`

`firmware/tools/pi_link/elena_link.py` drives the SPI link from the Pi for testing.

```sh
sudo apt install python3-spidev python3-libgpiod
python3 elena_link.py selftest              # no hardware: frame/CRC check against the spec examples
python3 elena_link.py status                # one decoded TELEMETRY
python3 elena_link.py poll -n 5000 -r 500   # POLL at 500 Hz, error statistics
python3 elena_link.py watch                 # live telemetry, 5 Hz
python3 elena_link.py list | get 1 | set 1 450 | save
python3 elena_link.py estop | clear | arm | drive 200 0
python3 elena_link.py badcrc | badlen | short | reboot
```

Options: `--hz` SPI clock (default 4 MHz), `--no-drdy` (fixed 1 ms wait instead of the DRDY pin),
`--drdy-gpio N` (default 7), `--chip /dev/gpiochipX` (auto-detected on the Pi 5). Supports libgpiod v1 and v2.

---

## Bench tests

[`firmware/docs/BENCH_TESTS.md`](firmware/docs/BENCH_TESTS.md) has a step-by-step test plan per firmware layer, with
expected console output, checkboxes and a results log. Run every test of a layer before moving to the next.

| Layer | Tests |
| --- | --- |
| L1 BSP | Boot banner and clock, LEDs, fault capture, IWDG |
| L2 RTOS + bus | Self-test, tasks/stack/CPU load, UART DMA, SPI3 chip selects, CAN TX/RX/filters/bus-off, watchdog under FreeRTOS |
| L3 devices | IMU readout and recovery, UNIDRIVE presence, SPEED_CMD/FEEDBACK, faults, parameters, LED blink codes |
| L4 Pi link | Link task and parameter block, flash save, first contact from the Pi, POLL stress, bad-frame rejection, parameters over SPI, robot commands, telemetry |

Common setup: bench supply (no Pi needed until L4), ST-Link on SWD, USB-UART on USART1 at 115200 8N1.

### F103 CAN tester / UNIDRIVE emulator

[`firmware/tools/can_tester_f103/`](firmware/tools/can_tester_f103/README.md) turns an STM32F103C8 + TJA1051T board
into a second CAN node for the L2 tests (ACK, logging, burst checks, echo mode for `canping`) and an emulator of
the four UNIDRIVE wheel drivers for L3, so the driver code can be tested before the real driver boards exist. Its
README covers wiring, CubeMX setup and commands.

---

## Repository layout

```
.
├── README.md
├── ELENA_Mk_1.pdf                    Schematic
├── ds-000347-icm-42688-p-v1.6.pdf    ICM-42688-P datasheet
└── firmware/
    ├── ELENA_Mk1_FW_V1.ioc           STM32CubeMX project
    ├── CMakeLists.txt
    ├── STM32H562xx_FLASH.ld          Linker script
    ├── Core/                         CubeMX-generated main.c and interrupt handlers
    ├── App/
    │   ├── Inc/config.h              Build-time switches
    │   ├── bsp/                      L1: reset, faults, LEDs, time, debug UART, watchdog
    │   ├── drivers/bus/              L2: UART DMA, SPI3, FDCAN, CRC, HAL callbacks
    │   ├── drivers/dev/              L3: ICM-42688-P, UNIDRIVE
    │   ├── comms/                    L4: SPI protocol and link, parameters
    │   ├── app/                      Tasks, shell, self-test, status LED, link glue
    │   └── os/                       FreeRTOSConfig.h, hooks
    ├── Middlewares/Third_Party/FreeRTOS/   FreeRTOS V11.1.0 kernel (vendored)
    ├── tests/host/                   PC unit tests (CTest)
    ├── tools/
    │   ├── pi_link/elena_link.py     Pi-side SPI link test tool
    │   └── can_tester_f103/          F103 CAN tester and UNIDRIVE emulator
    └── docs/BENCH_TESTS.md           Bench test plan and results
```

---

## Known issues and notes

- **J4 has no 3.3 V pin.** The ICM-42688-P is rated 3.6 V max; powering the IMU module from J4's +5V made it fail
  and also disturbed UART receive on the MCU. Feed the module from 3.3 V. Next revision: add +3V3 to J4.
- **J3 (SWD) has SWDIO, SWCLK and GND only**: no NRST and no 3.3 V reference pin.
- **No IMU interrupt line** on J4, so the IMU is polled at 1 kHz.
- **Watchdogs:** `CFG_IWDG_ENABLE` and `CFG_HOST_WDT_ENABLE` must both be 1 on the robot.
- **L5 (control) is not implemented yet**: the robot cannot be armed and motion commands over SPI are refused.
