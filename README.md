# Guition JC4880P443C_I_W (ESP32-P4 + ESP32-C6) — Field Notes & Troubleshooting

Hard-won notes for the **Guition JC4880P443C_I_W** 4.3″ touch display board
(module **JC-ESP32P4-M3**): **ESP32-P4** host + **ESP32-C6** Wi-Fi co-processor,
4.3″ **ST7701S** 480×800 MIPI-DSI panel, **GT911** I²C touch, **ES8311** audio,
microSD, 16 MB flash / 32 MB PSRAM.

## 📦 Vendor SDK / full board package

Guition's official download — demo code, drivers, datasheets, schematic, and
flashing tools for the JC4880P443C_I_W (~385 MB, 8-shard streamed download):

**<https://pan.jczn1688.com/directlink/1/HMI%20display/JC4880P443C_I_W.zip>**

(The `schematic/` sheets in this repo were extracted from it.)

The board ships with almost no English documentation, and three things bite
everyone who tries to use it outside the vendor's pre-built firmware:

1. **Wi-Fi associates then drops** (`ASSOC_LEAVE` / SDIO "dropping packets" loop)
2. **Display stays black** with the stock `esp_lcd_st7701` driver
3. **microSD won't mount** (`send_op_cond` 0x107 timeout)

All three + a full pin map are below. Verified on hardware (chip rev **v1.3**,
arduino-esp32 **3.3.6** / pioarduino `55.03.36-1`, ESP-IDF 5.5).

> Schematic sheets are in [`schematic/`](./schematic) (source: the
> [Reddit thread](https://www.reddit.com/r/esp32/comments/1nml6fm/guition_jc4880p433_43_capacitive_touch_display)
> + the vendor's `JC4880P443C_I_W.zip` download).

---

## 1. Wi-Fi: upgrade the ESP32-C6 co-processor firmware ⭐

**Symptom.** The P4 has no radio — Wi-Fi comes from the on-board **ESP32-C6** over
**SDIO (ESP-Hosted)**. Scanning works, but association is unstable:

```
H_SDIO_DRV: Dropping packet(s) from stream
H_SDIO_DRV: Failed to push data to rx queue
... Disconnected ... reason='Association Leave'  /  Reason: 8 - ASSOC_LEAVE
wifi: Restarting adapter        (loops forever; ~0/13 connects)
```

**Root cause.** A **host/slave version mismatch**. The board ships the C6 with
ESP-Hosted slave firmware **2.3.0**, but a modern arduino-esp32 / IDF host expects
~**2.12**. ESP-Hosted says so itself in the boot log:

```
transport: Version mismatch: Host [2.12.0] > Co-proc [2.3.0]
           ==> Upgrade co-proc to avoid RPC timeouts
```

It is **not** the AP, signal, power, app code, or the arduino-esp32 version — a
plain ESP32 (e.g. an M5 Core2) on the same network/app connects fine. Only the
P4→C6 ESP-Hosted link is affected.

### Fix — flash the C6 to a matching slave firmware (2.12.x)

Get the firmware from ESPHome's pre-built mirror (any `2.12.x` is fine):

```bash
curl -LO https://esphome.github.io/esp-hosted-firmware/v2.12.9/network_adapter_esp32c6.bin
```

The C6's UART0 + boot/reset are broken out on the **Expand-IO header (JP1)** — see
the schematic. Wire a **3.3 V USB-TTL adapter**:

| USB-TTL | → | JP1 (by label) |
|---------|---|----------------|
| TX      | → | `C6_U0RXD` |
| RX      | → | `C6_U0TXD` (crossover) |
| GND     | → | `GND` |
| 3V3     | ✗ | leave unconnected — the board powers the C6 |

> If your adapter has DTR/RTS auto-reset wired, esptool's `--before default_reset`
> enters download mode automatically. Otherwise pull **`C6_IO9`** (BOOT) low and
> pulse **`C6_CHIP_PU`** (reset).

**⚠️ CRITICAL — halt the P4 first.** While the P4 runs its app, its ESP-Hosted
stack keeps **resetting the C6 over GPIO54** (its WiFi-init retry). That resets
the C6 *mid-flash* → `A fatal error occurred: The chip stopped responding`
(small writes succeed, large ones die). Park the P4 in download mode first:

```bash
# 1) Halt the P4 (stops it from resetting the C6)
esptool --chip esp32p4 --port <P4_USB_PORT> --before default_reset --after no_reset flash_id

# 2) Flash the C6 (its app partitions are ota_0@0x10000 / ota_1@0x190000, 1.5 MB each)
esptool --chip esp32c6 --port /dev/<C6_UART> --baud 115200 \
        write_flash 0x10000 network_adapter_esp32c6.bin

# 3) Point the bootloader at the freshly-written app, then reset both
esptool --chip esp32c6 --port /dev/<C6_UART> erase_region 0xd000 0x2000   # otadata -> boots ota_0
esptool --chip esp32c6 --port /dev/<C6_UART> --after hard_reset flash_id
esptool --chip esp32p4 --port <P4_USB_PORT> --after hard_reset flash_id
```

**Result:** `Slave firmware version: 2.12.9`, `assoc_leaves=0`, associates cleanly,
DHCP + NTP + TLS all work. The update lives in the **C6's own flash**, so it
survives re-flashing the P4.

> A no-wire alternative (P4 OTAs the C6 over SDIO via
> `esp_hosted_slave_ota_*`) transfers fine but the ESPHome image fails the slave's
> `ota_end` validation (`OTA_VALIDATE_FAILED`); the direct UART flash above
> bypasses that and is the reliable path.

---

## 2. Display: ST7701S needs a MANUAL MIPI-DSI bring-up

**Symptom.** Black screen with the registry `esp_lcd_st7701` component.

**Fix.** Bring the panel up manually (this exact recipe works; credit:
[elik745i/ESP32-2432S024C-Remote](https://github.com/elik745i/ESP32-2432S024C-Remote)):

```
LDO chan 3 @ 2500 mV  (DSI-PHY power)
 → esp_lcd_new_dsi_bus     : 2 data lanes, 500 Mbps
 → esp_lcd_new_panel_io_dbi
 → esp_lcd_new_panel_dpi   : 480×800 RGB565, 34 MHz, h 12/42/42, v 2/8/166, num_fbs=1
 → panel reset GPIO5 (low 20 ms / high 120 ms)
 → custom ST7701 init table via esp_lcd_panel_io_tx_param
   (ends 0x11 SLPOUT +120 ms, 0x29 DISPON +20 ms)
 → esp_lcd_panel_init
 → backlight GPIO23 = plain digitalWrite(HIGH)   ← active-HIGH GPIO, NOT LEDC/PWM
```

Draw by writing the `get_frame_buffer()` framebuffer directly + `esp_cache_msync(…, C2M)`
per region. LVGL 8.3: `flush_cb` → framebuffer, `LV_COLOR_16_SWAP = 0`.

**Engineering-sample gotcha.** Rev v1.3 silicon is an **engineering sample**.
A `chip_variant: esp32p4` board def → *"Illegal instruction"* in the 2nd-stage
bootloader. Use **`chip_variant: esp32p4_es`** on a newer toolchain
(pioarduino `55.03.36-1`), **or** an older toolchain (arduino-esp32 **3.2.1** /
IDF 5.4.2 / pioarduino `54.03.21`) which boots the ES chip under plain `esp32p4`.

---

## 3. microSD: it's SDMMC, and you must power LDO VO4

**Symptom.** `sdmmc_init_ocr: send_op_cond returned 0x107` (timeout); the card
never initializes. SPI-mode `SD.begin()` with random pins just spins the
`sd_diskio` retry loop and **blocks boot**.

**Root cause.** The TF slot is **SDMMC (4-bit)**, not SPI — and `TF_VCC` is fed by
the P4 **on-chip LDO VO4** through an always-on P-FET (gate held low by R13; the
`GPIO45` enable is **unpopulated / NC**). If firmware never brings VO4 up, the
card has no power → `send_op_cond` times out. The data lines already have HW 5.1 kΩ
pull-ups (R11/R12), so pull-ups are not the issue.

**Fix.**

```c
// 1) Power TF_VCC: acquire on-chip LDO channel 4 @ 3.3 V (alongside the DSI's ch3)
esp_ldo_channel_config_t sd = { .chan_id = 4, .voltage_mv = 3300 };
esp_ldo_acquire_channel(&sd, &handle);

// 2) Mount over SDMMC 4-bit; tell SD_MMC NOT to manage an LDO itself
SD_MMC.setPins(43, 44, 39, 40, 41, 42);   // CLK, CMD, D0, D1, D2, D3
SD_MMC.setPowerChannel(-1);                // external power (we did VO4 above)
SD_MMC.begin("/sdcard", /*mode1bit=*/false, /*format=*/false);
```

> Don't let `SD_MMC` grab the default LDO channel — it collides
> (`esp_ldo_acquire_channel ... already in use / not adjustable`).

The SDMMC bus is **separate from the C6's Wi-Fi SDIO**, so SD + Wi-Fi coexist.

---

## 4. Hardware pin map

| Function | Pin(s) | Notes |
|----------|--------|-------|
| **Display** ST7701S | DSI lanes | 480×800, 2 lanes @ 500 Mbps, DPI 34 MHz |
| LCD reset | **GPIO5** | |
| LCD backlight | **GPIO23** | active-HIGH plain GPIO (not LEDC) |
| DSI-PHY power | on-chip **LDO ch3** 2500 mV | |
| **Touch** GT911 (I²C) | SDA **GPIO7** / SCL **GPIO8** | addr 0x5D; shared I²C w/ audio codec |
| Touch RST | **GPIO3** | (community docs also cite GPIO22) |
| **microSD** (SDMMC 4-bit) | CLK43 / CMD44 / D0-3 = 39/40/41/42 | HW 5.1 kΩ pull-ups |
| TF_VCC power | on-chip **LDO ch4** 3300 mV | `GPIO45` enable is **NC** |
| **Wi-Fi/BT** via C6 (SDIO) | CLK18 / CMD19 / D0-3 = 14/15/16/17 | 4-bit @ 40 MHz |
| C6 reset | **GPIO54** | active-HIGH |
| C6 UART0 (for flashing) | `C6_U0RXD` / `C6_U0TXD` on **JP1** | + `C6_IO9` (BOOT), `C6_CHIP_PU` (reset) |
| **Audio** ES8311 (I²S) | MCLK13 / BCLK12 / LRCK10 / DOUT9 / DIN48 | amp PA enable GPIO11 |
| Codec I²C | GPIO7 / GPIO8 | shared with touch |
| Boot button | GPIO35 | |
| Builtin LED | GPIO26 | also RS485 UART1-TX in the RS485 build |
| UART0 console | TXD37 / RXD38 | |
| RS485 (MAX485) | UART1 TX26 / RX27 | |
| USB | High-Speed = P4 (`ESP_USB`) · Full-Speed = `USB1P1` | see sheet 03 |
| Power | 3V3 via TLV62569 buck · battery via IP5306 | |

Two on-chip LDO channels are used: **ch3 = DSI-PHY** (2.5 V), **ch4 = TF_VCC/SD** (3.3 V).

---

## Schematic

| Sheet | Contents |
|-------|----------|
| [01](./schematic/01-audio-es8311_sd-card_wifi-antenna.webp) | ES8311 codec · **TF-card + TF_VCC LDO (VO4)** · C6 Wi-Fi antenna |
| [02](./schematic/02-speaker_power-3v3_uart-input_battery.webp) | NS4150 speaker amp · TLV62569 3V3 · UART input · IP5306 battery |
| [03](./schematic/03-expand-io_usb-fs-hs_lcd-backlight.webp) | **Expand-IO header (JP1, incl. C6 UART/boot)** · USB FS/HS · MP3202 backlight |
| [04](./schematic/04-rs485-max485.webp) | MAX485 RS485 transceiver |
| [05](./schematic/05-adc-mic.webp) | MSM381 analog mic (ADC/AEC) |

---

## Sources & credits

- Schematic + vendor demo: the
  [Reddit thread](https://www.reddit.com/r/esp32/comments/1nml6fm/guition_jc4880p433_43_capacitive_touch_display)
  and Guition's `JC4880P443C_I_W.zip` download.
- ST7701 manual bring-up: [elik745i/ESP32-2432S024C-Remote](https://github.com/elik745i/ESP32-2432S024C-Remote).
- C6 slave firmware mirror: [esphome/esp-hosted-firmware](https://github.com/esphome/esp-hosted-firmware).
- P4↔C6 + SD reference: [buccaneer-jak/JC4880P443C-...](https://github.com/buccaneer-jak/JC4880P443C-P4-to-C6-RS232-communication-and-SD-Card-connection).
- Pin-map cross-check: [seobaeksol/JC4880P443C_I_W](https://github.com/seobaeksol/JC4880P443C_I_W).
- ESP-Hosted slave OTA mechanism: [espressif/esp-hosted-mcu](https://github.com/espressif/esp-hosted-mcu).

Issues / corrections welcome.
