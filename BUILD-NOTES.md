# Build & Flash Notes (ESP32-DIV-Creepster)

This is a modified fork of [cifertech/ESP32-DIV](https://github.com/cifertech/ESP32-DIV)
with the menu UI restyled in the **Creepster** font and a **"The Creepster"**
boot splash. Because the source was changed, you must **compile it yourself** —
the official web flasher and any prebuilt `.bin` would overwrite these changes
with the stock firmware.

The board is already set to **ESP32-DIV V2** (ESP32-S3) in
`ESP32-DIV/BoardConfig.h`, so no board edit is needed.

---

## 1. Get the code — and actually extract it

Download the repo ZIP (green **Code** button → **Download ZIP**).

**Do not open the `.ino` from inside the ZIP.** Double-clicking a file inside
the Windows ZIP preview extracts only that one file to a temp folder, so the
sketch loses its 30+ sibling files and you get errors like:

```
fatal error: SettingsStore.h: No such file or directory
```

Instead: **right-click the ZIP → "Extract All…"**, extract to a normal folder
(e.g. `Documents`), then open `ESP32-DIV/ESP32-DIV.ino` from the **extracted**
folder. Confirm `SettingsStore.h`, `wifi.cpp`, `Creepster.h`, `splash.h`, etc.
all sit next to the `.ino`.

## 2. ESP32 board package

Arduino IDE → **Tools → Board → Boards Manager** → search **esp32** → install
**"esp32 by Espressif Systems."**

## 3. Libraries (Tools → Manage Libraries)

`TFT_eSPI`, `XPT2046_Touchscreen`, `PCF8574`, `ArduinoJson`, `NimBLE-Arduino`,
`IRremoteESP8266`, `RF24`, `RCSwitch`, `ELECHOUSE_CC1101`, `Adafruit PN532`.

(`Update`, `WebServer`, `Preferences`, `EEPROM`, `DNSServer`, `ESPmDNS`, `SD`
ship with the ESP32 core.)

> **TFT_eSPI must be configured for the V2 display** or the screen stays blank.
> See the upstream [Wiki](https://github.com/cifertech/ESP32-DIV/wiki) for the
> correct display setup.

## 4. Board settings (Tools menu)

V2 is an **ESP32-S3**:

- **Board:** ESP32S3 Dev Module
- **PSRAM:** OPI PSRAM
- **USB CDC On Boot:** Enabled
- **Flash Size:** 16MB (match your module)
- **Partition Scheme:** one with enough app space (e.g. 16M Flash, 3MB APP)

Match the exact values from the upstream Wiki if in doubt.

---

## 5. Fix the linker error (REQUIRED)

On compile you will hit:

```
multiple definition of `ieee80211_raw_frame_sanity_check';
... first defined here
collect2.exe: error: ld returned 1 exit status
```

**This is not caused by the Creepster changes** — the stock ESP32-DIV firmware
hits it too. The firmware deliberately redefines
`ieee80211_raw_frame_sanity_check()` (in `wifi.cpp`) to disable the ESP32's
built-in block on raw-frame injection, which is what powers the deauther /
beacon features. On ESP32 core 2.0.x the linker refuses the duplicate symbol.

**Fix — add the `-zmuldefs` linker flag** (lets the linker accept the override
and keep deauth working):

1. Close the Arduino IDE.
2. Open (adjust the version folder to match what you installed):
   ```
   C:\Users\<you>\AppData\Local\Arduino15\packages\esp32\hardware\esp32\2.0.10\platform.txt
   ```
3. Find the single long line that **starts with** `compiler.c.elf.flags.esp32s3=`
   (it begins `-T memory.ld ...` and ends `... -u __cxa_guard_dummy`). Do **not**
   edit the `.esp32`, `.esp32s2`, or `.esp32c3` variants.
4. At the very **end** of that line, add a space then:
   ```
   -Wl,-zmuldefs
   ```
5. Save, reopen the Arduino IDE, compile again.

Notes:
- The `Multiple libraries were found for "SD.h"` message is a harmless
  **warning**, not the error — the core's SD library is used.
- This edit applies to all ESP32-S3 builds and is **wiped if you update or
  reinstall the ESP32 core** — just re-add the flag if that happens.

---

## 6. Flash it — two options

- **Over USB:** plug in, select the **Port** under Tools, click **Upload**.
  Hold the **BOOT** button if it won't enter download mode.
- **From SD (no PC after building):** Arduino IDE →
  **Sketch → Export Compiled Binary**, rename the output to **`firmware.bin`**,
  copy it to the **root of the SD card**, insert it, then on the device go to
  **Tools → Update Firmware**.

On boot you'll see the ~3-second "The Creepster" splash, then the Creepster menus.

---

## Troubleshooting

- **Splash colors look red/blue-swapped:** in `ESP32-DIV/ESP32-DIV.ino`, change
  `tft.setSwapBytes(true);` (in `setup()`, around the splash `pushImage`) to
  `tft.setSwapBytes(false);` and reflash.
- **Splash too long/short:** change `delay(3000);` next to the splash in
  `setup()`.
- **Blank screen:** TFT_eSPI isn't configured for the display (see step 3).
