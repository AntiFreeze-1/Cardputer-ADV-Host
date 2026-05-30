# Cardputer ADV ↔ Pi Zero W — Graphical Display Bridge

### Firmware Project Handoff Document

-----

## Project Goal

Build firmware for both sides of a hardware bridge that:

- Forwards the Pi Zero W’s desktop framebuffer to the M5Stack Cardputer ADV’s display over SPI GPIO
- Forwards Cardputer keyboard input back to the Pi over UART, injected as Linux input events

The end result is a Pi Zero W desktop (240×135) controllable entirely from the Cardputer ADV.

-----

## Hardware

|Device               |Role                                  |
|---------------------|--------------------------------------|
|M5Stack Cardputer ADV|Display output + keyboard input device|
|Raspberry Pi Zero W  |Runs Raspberry Pi OS, host system     |

### Cardputer ADV Specs

- MCU: ESP32-S3FN8 (Stamp-S3A module)
- RAM: 512KB — **no PSRAM**
- Display: ST7789, 240×135, driven internally over SPI
- Keyboard: 56-key matrix
- Expansion: EXT 2.54-14P header (primary connection point)
- Dev environment: **PlatformIO + Arduino framework**
- Libraries available: M5GFX, M5Unified

### Pi Zero W Specs

- CPU: ARM11 single-core
- OS: Raspberry Pi OS (32-bit)
- I/O: GPIO SPI0, GPIO UART (ttyAMA0 preferred)
- Note: Disable Bluetooth (`dtoverlay=disable-bt`) to free ttyAMA0 — do NOT use ttyS0 (mini UART, unstable)

-----

## Physical Connection

All wiring goes through the **EXT 2.54-14P expansion header** on the Cardputer ADV.
Exact GPIO pin numbers on the ESP32-S3 side must be verified against the official Cardputer ADV schematic before finalizing.

```
Pi Zero W GPIO          Cardputer ADV EXT Header
────────────────────────────────────────────────
SPI0 MOSI (GPIO10)  ──► ESP32 SPI slave MOSI
SPI0 SCLK (GPIO11)  ──► ESP32 SPI slave SCLK
SPI0 CE0  (GPIO8)   ──► ESP32 SPI slave CS
GPIO (any, e.g. 25) ──► ESP32 DC/frame-sync pin  (signals start of new frame)
UART TX   (GPIO14)  ──► ESP32 UART RX
UART RX   (GPIO15)  ◄── ESP32 UART TX
GND                 ──► GND
3.3V                ──► 3.3V (check current budget vs Cardputer battery output)
```

-----

## Architecture

### Display Path (Pi → Cardputer)

```
Pi framebuffer (/dev/fb1, 240×135 RGB565)
        │
        │  SPI GPIO (~20MHz)
        ▼
ESP32-S3 SPI slave receives frame into Buffer B
        │
        │  When frame complete, swap buffers
        ▼
DMA SPI pushes Buffer A → ST7789 display
(CPU free during DMA transfer — keyboard still scans)
```

- Full frame = 240 × 135 × 2 bytes = **64,800 bytes (~63KB)**
- Double buffer = ~126KB total — fits in 512KB with careful heap management
- Use **DMA SPI** (M5GFX supports this) so display writes are non-blocking
- SPI framing protocol: Pi sends a magic header (e.g. `0xABCD` + 2-byte length) before each frame

### Input Path (Cardputer → Pi)

```
Cardputer key matrix scan
        │
        │  UART (115200+ baud)
        ▼
Pi UART daemon reads keypress packets
        │
        │  /dev/uinput
        ▼
Linux input subsystem (Pi sees a virtual keyboard)
```

- Packet format suggestion: `[0xFF] [keycode] [state: 0x01 press / 0x00 release]`
- Keycodes should map to Linux evdev key codes
- Fn key combos (arrow keys, special chars) should be resolved on the Cardputer side before sending

-----

## Pi Side — Files to Produce

### 1. Device Tree Overlay — `st7789-cardputer.dts`

- Defines a 240×135 ST7789 SPI display on SPI0
- Loaded via `dtoverlay=` in `/boot/config.txt`
- Use `panel-mipi-dbi` driver (preferred on newer kernels) or `fbtft/fb_st7789v`
- Include correct init sequence for ST7789 at 240×135

### 2. `/boot/config.txt` additions

```
dtparam=spi=on
enable_uart=1
dtoverlay=disable-bt
dtoverlay=st7789-cardputer
framebuffer_width=240
framebuffer_height=135
```

### 3. uinput keyboard daemon — `kbd_daemon.c`

- Opens `/dev/ttyAMA0` at 115200 baud
- Reads 3-byte packets `[0xFF] [keycode] [state]`
- Opens `/dev/uinput` and registers a virtual keyboard device
- Translates received keycodes to `EV_KEY` events via uinput
- Should run as a systemd service on boot

### 4. Systemd service — `kbd-daemon.service`

- Starts `kbd_daemon` after `multi-user.target`
- Restart on failure

-----

## Cardputer Side — Files to Produce

### 1. `platformio.ini`

```ini
[env:cardputer-adv]
platform = espressif32
board = esp32-s3-devkitc-1
framework = arduino
upload_speed = 1500000
monitor_speed = 115200
build_flags =
    -DESP32S3
    -DARDUINO_USB_CDC_ON_BOOT=1
    -DARDUINO_USB_MODE=1
lib_deps =
    m5stack/M5GFX
    m5stack/M5Unified
```

### 2. `src/main.cpp` — Main firmware

Responsibilities:

- Initialize M5GFX display
- Initialize SPI slave on EXT header pins (verify pin numbers from schematic)
- Initialize UART for keyboard TX
- Main loop: receive frame over SPI slave → swap buffers → trigger DMA push to display
- Keyboard matrix scan → encode → send over UART

### 3. `src/spi_slave.h` / `src/spi_slave.cpp`

- ESP32-S3 SPI slave configuration
- Frame sync via DC/frame-sync GPIO pin
- ISR or DMA-based receive into double buffer
- Frame-complete callback

### 4. `src/display.h` / `src/display.cpp`

- M5GFX wrapper
- `pushFrameDMA(uint8_t* buf)` — non-blocking DMA push to ST7789
- Buffer swap logic
- `onDMAComplete` callback to signal ready for next frame

### 5. `src/keyboard.h` / `src/keyboard.cpp`

- Cardputer key matrix scanning (reference M5Stack Cardputer keyboard library for matrix layout)
- Fn-layer handling (map Fn+key combos to special keycodes)
- `encodePacket(keycode, state)` → 3-byte packet
- UART TX send

-----

## Key Constraints & Gotchas

- **No PSRAM** — every allocation counts. Two 63KB framebuffers + stack + libs must fit in 512KB. Allocate framebuffers statically, not on heap.
- **DMA is mandatory** — blocking SPI writes to the display will freeze keyboard scanning. Use M5GFX’s DMA path.
- **ttyAMA0 not ttyS0** — the Pi mini UART is slaved to the GPU clock and unreliable. Always disable BT and use the full PL011 UART.
- **SPI framing** — without a clear frame delimiter, the ESP32 won’t know where one frame ends and the next begins. Use a magic byte header + length, or use the dedicated DC GPIO pin as a frame-start strobe.
- **Pi SPI speed** — start at 10–20MHz and validate signal integrity before pushing higher. At 20MHz a full frame transfers in ~26ms (~38fps theoretical).
- **Cardputer ADV pinout** — the EXT 2.54-14P header pin-to-GPIO mapping must be verified against the official M5Stack schematic at <https://docs.m5stack.com/en/core/Cardputer-Adv> before any pin assignments are finalized in code.

-----

## Suggested Build Order

1. `platformio.ini` + project skeleton
1. Pi device tree overlay + `/boot/config.txt`
1. Cardputer SPI slave receive (loopback test — just print received bytes over USB serial)
1. Cardputer DMA display push (test with a static color fill)
1. Pi → Cardputer full frame transfer (static desktop screenshot as test)
1. Cardputer keyboard scan + UART TX
1. Pi uinput daemon + systemd service
1. Full integration test

-----

## Reference Links

- Cardputer ADV docs: <https://docs.m5stack.com/en/core/Cardputer-Adv>
- M5GFX (display + DMA): <https://github.com/m5stack/M5GFX>
- ESP32-S3 SPI slave (ESP-IDF reference): <https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/peripherals/spi_slave.html>
- Linux uinput: <https://www.kernel.org/doc/html/latest/input/uinput.html>
- panel-mipi-dbi driver: <https://github.com/notro/panel-mipi-dbi>