<p align="right">
  <strong>简体中文</strong> · <a href="README.md">English</a>
</p>

# FoloToy AI Passport 固件切换器

AI Passport 出厂是一机一固件：刷一个玩法、用完再刷下一个。这个项目把它
变成**多固件切换器**——把多个固件留在设备里，开机时选哪个就进哪个。

## 为什么做这个

- 一台设备装多个固件：平时开机直接进入上次用的固件。
- 开机时按住任意按键约 2 秒进入切换菜单（bootloader 标准 factory-reset
  引脚，不擦任何数据分区）。
- 菜单显示每个槽位固件的版本和构建日期，标注“上次”，选中后写入启动
  选择并重启进入。
- 菜单 10 秒无操作自动启动上次固件，误入也不慌。
- 全程离线，无需任何配网。

## 两个版本，同一种玩法

| | 三槽版 | 双槽版 |
| --- | --- | --- |
| 槽位 | A / B / C | A / B |
| 每槽上限 | **2MB** | **约 3.56MB**（3,735,552 字节） |
| 槽位偏移 | A `0x120000` · B `0x320000` · C `0x520000` | A `0xE0000` · B `0x470000` |
| 装得下原厂固件（约 3.52MB）？ | 不能 | **能**（余量约 47KB） |
| 固件文件 | [`firmware/3-slot/FoloToy-AI-Passport-full.bin`](firmware/3-slot/FoloToy-AI-Passport-full.bin) | [`firmware/dual-slot/FoloToy-AI-Passport-full.bin`](firmware/dual-slot/FoloToy-AI-Passport-full.bin) |
| 源码 | [`source/three-slot-source.zip`](source/three-slot-source.zip) | [`source/dual-slot-source.zip`](source/dual-slot-source.zip) |

所有文件的校验和见 [`firmware/SHA256SUMS`](firmware/SHA256SUMS)。

## 刷入切换器

```bash
pip install esptool

# 三槽版:
esptool --chip esp32c3 -p /dev/cu.usbmodem* -b 921600 \
    write_flash --verify 0x0 firmware/3-slot/FoloToy-AI-Passport-full.bin

# 双槽版:
esptool --chip esp32c3 -p /dev/cu.usbmodem* -b 921600 \
    write_flash --verify 0x0 firmware/dual-slot/FoloToy-AI-Passport-full.bin
```

合并镜像包含 bootloader、分区表、切换器本体，以及预装在槽A 的硬件测试
应用；其余槽位保持为空，直到你刷入固件。

## 往槽位里装固件

槽位只接受**纯 app 镜像**（文件以 `0xE9` 魔数开头），单个不超过上表限制。

```bash
# 双槽示例:把固件写入槽B(偏移 0x470000)
esptool --chip esp32c3 -p /dev/cu.usbmodem* -b 921600 \
    write_flash --verify 0x470000 你的固件.bin
```

玩法社区下载的合并镜像（`esp-merged-0x0`）不是 app 镜像，需要先提取；
现成的 Python 提取片段在教程里。

## 使用

1. 开机 → 直进上次固件（首次开机进选择菜单）。
2. 开机按住任意按键约 2 秒 → 进入切换菜单。
3. 上/下移动，OK 启动选中固件。
4. 10 秒无操作自动启动上次固件。

## 教程与源码构建

- [三槽版教程](docs/tutorial-3slot.zh_CN.md) · [Dual-Slot tutorial](docs/tutorial-dual.md)
- [`source/`](source/) 下的源码包是完整的修改后源码树（基线为上游
  [FoloToy/ai-passport](https://github.com/FoloToy/ai-passport)，MIT）。
- 从源码构建需要 ESP-IDF 5.5.3，完整步骤见教程。

## 许可

MIT。上游基线：[FoloToy/ai-passport](https://github.com/FoloToy/ai-passport)
（[Gitee 镜像](https://gitee.com/FoloToy/ai-passport)）。
