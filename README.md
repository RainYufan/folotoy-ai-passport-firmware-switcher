<p align="right">
  <a href="README.zh_CN.md">简体中文</a> · <strong>English</strong>
</p>

# FoloToy AI Passport 固件切换器 · FoloToy AI Passport Firmware Switcher

The AI Passport ships as a one-firmware device — flash a play, use it, and
flash again to try something else. This project turns it into a
**multi-firmware switcher**: keep several firmwares on the device and choose
which one boots.

## Why

- One device, several firmwares: normal power-on boots your last-used firmware
  directly.
- Hold any button at power-on for about 2 seconds to enter the switch menu
  (standard bootloader factory-reset pin; it never erases your data
  partitions).
- The menu shows each slot's firmware version and build date, marks the last
  used one, and boots your choice after writing the boot selection.
- 10 seconds of idle auto-boots the last-used firmware, so an accidental entry
  recovers by itself.
- Fully offline — no network setup of any kind.

## Two editions, one idea

| | 3-Slot Edition | Dual-Slot Edition |
| --- | --- | --- |
| Slots | A / B / C | A / B |
| Size limit per slot | **2 MB** each | **≈3.56 MB** each (3,735,552 bytes) |
| Slot offsets | A `0x120000` · B `0x320000` · C `0x520000` | A `0xE0000` · B `0x470000` |
| Fits the stock system firmware (≈3.52 MB)? | No | **Yes** (with ≈47 KB margin) |
| Firmware file | [`firmware/3-slot/FoloToy-AI-Passport-full.bin`](firmware/3-slot/FoloToy-AI-Passport-full.bin) | [`firmware/dual-slot/FoloToy-AI-Passport-full.bin`](firmware/dual-slot/FoloToy-AI-Passport-full.bin) |
| Source | [`source/three-slot-source.zip`](source/three-slot-source.zip) | [`source/dual-slot-source.zip`](source/dual-slot-source.zip) |

Checksums for every file are in [`firmware/SHA256SUMS`](firmware/SHA256SUMS).

## Flash the switcher

```bash
pip install esptool

# 3-slot edition:
esptool --chip esp32c3 -p /dev/cu.usbmodem* -b 921600 \
    write_flash --verify 0x0 firmware/3-slot/FoloToy-AI-Passport-full.bin

# dual-slot edition:
esptool --chip esp32c3 -p /dev/cu.usbmodem* -b 921600 \
    write_flash --verify 0x0 firmware/dual-slot/FoloToy-AI-Passport-full.bin
```

The merged image includes the bootloader, partition table, the switcher, and a
preloaded hardware-test app in slot A; the remaining slots stay empty until you
flash them.

## Flash firmware into slots

Use the companion desktop tool
[ai-passport-slot-flasher](https://github.com/RainYufan/ai-passport-slot-flasher):
pick a firmware and a slot, and it writes only that slot — never the switcher
itself. Installers (Windows `.exe`, macOS `.dmg`, Linux `.tar.gz`) are on its
[Releases](https://github.com/RainYufan/ai-passport-slot-flasher/releases).

## Extract the app image first (important)

The play store serves `esp-merged-0x0` full images, but a slot only accepts a
**plain app image** (file starts with the `0xE9` magic byte). Use the
**extract-play** tool from the same repo:

- GUI `extract_play_gui`: paste the play link, or drag a merged `.bin` into the
  window, click "提取 app 镜像" to get `xxx-app.bin`.
- CLI: `python extract_play.py "<play link>"` (Windows: double-click
  `extract_play.exe`).

A manual Python snippet is in the tutorials — the key rule is **copy the whole
app region, do not trim**, or you cut the trailing digest and the image won't
boot.

## Put a firmware into a slot

With `xxx-app.bin` ready, flash it with esptool to the matching offset:

```bash
# dual-slot example: write a firmware to slot B (offset 0x470000)
esptool --chip esp32c3 -p /dev/cu.usbmodem* -b 921600 \
    write_flash --verify 0x470000 xxx-app.bin
```

## Use

1. Power on → boots the last-used firmware (first boot opens the menu).
2. Hold any button at power-on, keep holding ≈2 s → the switch menu.
3. UP/DOWN move, OK launches the selected firmware.
4. 10 s idle auto-boots the last-used firmware.

## Tutorials & build-from-source

- [3-Slot tutorial](docs/tutorial-3slot.md) · [双槽教程](docs/tutorial-dual.zh_CN.md)
- Source archives under [`source/`](source/) contain the complete modified
  source trees (baseline: upstream
  [FoloToy/ai-passport](https://github.com/FoloToy/ai-passport), MIT).
- Building requires ESP-IDF 5.5.3; see the tutorials for the full steps.

## License

MIT. Upstream baseline: [FoloToy/ai-passport](https://github.com/FoloToy/ai-passport)
([Gitee mirror](https://gitee.com/FoloToy/ai-passport)).
