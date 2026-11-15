<p align="right">
  <a href="firmware-selector-tutorial.zh_CN.md">简体中文</a> · <strong>English</strong>
</p>

# Boot Firmware Selector Tutorial

A step-by-step guide for the rec-style boot firmware selector built on the
[FoloToy AI Passport](https://ai-passport.folotoy.cn/en/) open platform
(ESP32-C3 · 8 MB Flash · 240×320 display · three buttons).

## 1. What it is

A small launcher firmware that turns the device into a firmware chooser:

- Normal power-on boots the **last selected firmware** directly.
- Holding any button (OK works) at power-on for about **2 seconds** enters the
  selector menu (standard ESP-IDF bootloader factory-reset pin on GPIO0; it
  erases only the OTA data partition, never your data partitions).
- The menu lists up to **three firmware slots (A/B/C, 2 MB each)**, shows each
  image's version and build date, marks the last used one, and boots the chosen
  image after writing the boot selection.
- After 10 s of inactivity the menu auto-boots the last used firmware, so an
  accidental entry recovers by itself.

## 2. File manifest

Everything that makes up this project, relative to the upstream `main` baseline:

| Path | Role | Status |
| --- | --- | --- |
| [main/main.c](/main/main.c) | Selector application: init, slot discovery, boot action | rewritten |
| [main/boot_selector_model.c](/main/boot_selector_model.c), [.h](/main/boot_selector_model.h) | Pure menu state machine (no ESP-IDF/LVGL deps) | added |
| [main/boot_ui.c](/main/boot_ui.c), [.h](/main/boot_ui.h) | Redesigned dark-list UI layer | added |
| [assets/fonts/app_font_14.c](/assets/fonts/app_font_14.c), [app_font_20.c](/assets/fonts/app_font_20.c) | Noto Sans SC subset fonts (generated) | added |
| [tools/gen_fonts.sh](/tools/gen_fonts.sh) | Reproducible font generation | added |
| [tools/build_hwtest_payload.sh](/tools/build_hwtest_payload.sh) | Rebuilds the slot-A hardware-test payload | added |
| [payloads/hwtest-app.bin](/payloads/hwtest-app.bin), [README.md](/payloads/README.md) | Prebuilt slot-A image + provenance | added |
| [tests/test_boot_selector_model.c](/tests/test_boot_selector_model.c) | Host tests for the menu model | added |
| [tests/test_partition_layout.py](/tests/test_partition_layout.py) | Partition-table contract test | added |
| [partitions.csv](/partitions.csv) | nvs/phy/factory/otadata + slot_a/b/c layout | rewritten |
| [sdkconfig.defaults](/sdkconfig.defaults) | Factory-reset pin, hold time, OTA-data erase | extended |
| [main/CMakeLists.txt](/main/CMakeLists.txt) | Selector sources + slot-A merge target | rewritten |
| [tools/validate.sh](/tools/validate.sh) | Registers the two new host tests | extended |
| [assets/README.md](/assets/README.md) | Font provenance records | extended |
| [README.md](/README.md) | Project description and usage | rewritten |
| [docs/firmware-selector-tutorial.md](/docs/firmware-selector-tutorial.md) | This tutorial | added |

Untouched: everything under `components/bsp/`, the baseline hardware-test
sources (`main/demo_*.c`, `main/ui_pixel.*` — kept for the host tests and the
slot-A payload build, not compiled into the selector).

## 3. Flash the selector

Prerequisites: a data-capable USB-C cable and `esptool`
(`pip install esptool`).

```bash
# Fresh install (writes bootloader + partition table + selector + slot A):
esptool --chip esp32c3 -p /dev/cu.usbmodem1101 -b 921600 \
    write_flash --verify 0x0 build/FoloToy-AI-Passport-full.bin
```

The merged image only spans `0x0..0x294600`; slots B and C stay empty until you
flash them.

## 4. Put a firmware into a slot

Slots accept **app-only images** (starting with the `0xE9` magic), each at most
2 MB:

| Slot | Flash offset | esptool example |
| --- | --- | --- |
| A | `0x120000` | `write_flash 0x120000 your-app.bin` |
| B | `0x320000` | `write_flash 0x320000 your-app.bin` |
| C | `0x520000` | `write_flash 0x520000 your-app.bin` |

The community play store serves `esp-merged-0x0` archives, which cannot go into
a slot directly. Extract the app image from the merged archive first, for
example:

```python
import struct
data = open("play-merged.bin", "rb").read()
tbl = data[0x8000:0x9000]
ENTRY = struct.Struct("<HBBII16sI")
off = 0
while off + ENTRY.size <= len(tbl):
    magic = int.from_bytes(tbl[off:off+2], "little")
    if magic in (0xFFFF, 0xEBEB):
        break
    _, kind, _, poff, size, _, _ = ENTRY.unpack_from(tbl, off)
    if kind == 0:  # app partition
        open("extracted-app.bin", "wb").write(data[poff:poff+size])
        break
    off += ENTRY.size
```

Then verify and flash:

```bash
esptool image_info extracted-app.bin        # must print a valid ESP32-C3 image
esptool --chip esp32c3 -p /dev/cu.usbmodem1101 -b 921600 \
    write_flash --verify 0x320000 extracted-app.bin
```

The selector reads each image's app description (project name, version, date)
and lists only valid images.

## 5. Usage

| Action | Result |
| --- | --- |
| Power on | Boots the last selected firmware (selector menu on first boot) |
| Hold any button at power-on, keep holding ~2 s | Enters the selector menu |
| UP / DOWN | Move the selection (cancels the countdown) |
| OK (short press) | Boots the selected firmware |
| No input for 10 s | Auto-boots the last used firmware |

Battery level is shown in the top-right corner (`--` when unavailable).

## 6. Build from source

1. Install ESP-IDF **5.5.3** for ESP32-C3
   ([official guide](https://docs.espressif.com/projects/esp-idf/en/v5.5.3/esp32c3/get-started/linux-macos-setup.html)),
   plus `cmake` and `ninja` on your PATH.
2. Activate and build:

   ```bash
   . $HOME/esp/esp-idf-v5.5.3/export.sh
   idf.py set-target esp32c3
   idf.py build
   ```

3. Rebuild the slot-A payload and the merged image:

   ```bash
   ./tools/build_hwtest_payload.sh
   ./tools/validate.sh --firmware   # outputs build/FoloToy-AI-Passport-full.bin
   ```

4. If you change any UI text, regenerate the font subsets (lv_font_conv 1.5.3 +
   a licensed CJK TTF such as Noto Sans SC) and re-run the glyph gate:

   ```bash
   ./tools/gen_fonts.sh
   ./tools/validate.sh --static
   ```

## 7. Customize

- **Slot count / sizes**: edit [partitions.csv](/partitions.csv), keep app
  partitions 64 KB aligned and inside 8 MB, and mirror the constants in
  [main/boot_selector_model.h](/main/boot_selector_model.h) and
  [tests/test_partition_layout.py](/tests/test_partition_layout.py).
- **Menu entry hold time**: `CONFIG_BOOTLOADER_HOLD_TIME_GPIO` in
  [sdkconfig.defaults](/sdkconfig.defaults).
- **Menu text**: edit the string literals in `main/boot_ui.c` / `main/main.c`,
  then run [tools/gen_fonts.sh](/tools/gen_fonts.sh) and the static gate — the
  glyph checker fails the build if any new character is missing from the fonts.

## 8. FAQ

**Both slots look identical in the menu.** Many apps built from this repository
share the project name; the slot card leads with `v<version> · <date>`, which is
unique per build.

**Holding the button at power-on does not open the menu.** Keep the button held
for the full hold time (2 s). All three buttons share one ADC line, so any
button works at the bootloader level.

**A slot shows the "empty slot" label.** Either nothing is flashed there or the image is not
a valid app image. Re-check with `esptool image_info`.

**The selected firmware did not boot.** The bootloader validates the image and
falls back to the selector menu when validation fails. Re-flash the slot and
verify.

**I want my original firmware back.** This project replaces the original
firmware; the original (RootPassport, ~3.52 MB + a 3 MB voice partition) does
not fit the 2 MB slots. Flash any app-image firmware into a slot instead.

## 9. Open-source notes

- License: MIT (the repository [LICENSE](/LICENSE)); the upstream baseline is
  [FoloToy/ai-passport](https://gitee.com/FoloToy/ai-passport)
  ([GitHub mirror](https://github.com/FoloToy/ai-passport)).
- Fonts: Noto Sans SC subsets, SIL OFL 1.1; provenance is recorded in
  [assets/README.md](/assets/README.md).
- The prebuilt `payloads/hwtest-app.bin` is rebuilt deterministically by
  [tools/build_hwtest_payload.sh](/tools/build_hwtest_payload.sh) from the
  baseline commit.
- Validation status of this delivery: Build PASS, Host tests PASS,
  Device tests PASS (boot, menu, slot discovery, OTA-partition switching
  verified on a real device), `actionlint` not run (offline environment).
