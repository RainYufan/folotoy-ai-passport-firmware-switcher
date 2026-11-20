<p align="right">
  <strong>简体中文</strong> · <a href="firmware-selector-tutorial.md">English</a>
</p>

# 固件切换器教程

基于 [FoloToy AI Passport](https://ai-passport.folotoy.cn/) 开放平台
(ESP32-C3 · 8 MB Flash · 240×320 屏 · 三按键)的 rec 式固件启动切换器,
从零到开源的完整说明。

## 1. 这是什么

一个把设备变成"固件选择器"的启动器固件:

- 正常开机直接进入**上次选择的固件**。
- 开机时按住任意按键(OK 即可)约 **2 秒**进入选择器菜单——这是 ESP-IDF
  bootloader 标准的 factory-reset 引脚机制(GPIO0 低电平),只擦 OTA 数据
  分区,不动任何数据分区。
- 菜单列出最多**三个固件槽(A/B/C,各 2MB)**,显示每个镜像的版本与构建
  日期,标注上次使用的槽位;选中后写入启动选择并重启进入。
- 菜单空闲 10 秒自动启动上次固件,误入菜单也能自行恢复。

## 2. 文件清单

相对上游 `main` 基线,本项目的全部变更:

| 路径 | 作用 | 状态 |
| --- | --- | --- |
| [main/main.c](/main/main.c) | 选择器应用:初始化、槽位扫描、启动动作 | 重写 |
| [main/boot_selector_model.c](/main/boot_selector_model.c)、[.h](/main/boot_selector_model.h) | 纯逻辑菜单状态机(不依赖 ESP-IDF/LVGL) | 新增 |
| [main/boot_ui.c](/main/boot_ui.c)、[.h](/main/boot_ui.h) | 全新设计的深色列表界面层 | 新增 |
| [assets/fonts/app_font_14.c](/assets/fonts/app_font_14.c)、[app_font_20.c](/assets/fonts/app_font_20.c) | Noto Sans SC 子集字体(生成产物) | 新增 |
| [tools/gen_fonts.sh](/tools/gen_fonts.sh) | 可复现的字体生成脚本 | 新增 |
| [tools/build_hwtest_payload.sh](/tools/build_hwtest_payload.sh) | 重建槽A 硬件测试载荷 | 新增 |
| [payloads/hwtest-app.bin](/payloads/hwtest-app.bin)、[README.md](/payloads/README.md) | 预编译槽A 镜像 + 来源说明 | 新增 |
| [tests/test_boot_selector_model.c](/tests/test_boot_selector_model.c) | 菜单状态机宿主机测试 | 新增 |
| [tests/test_partition_layout.py](/tests/test_partition_layout.py) | 分区表契约测试 | 新增 |
| [partitions.csv](/partitions.csv) | nvs/phy/factory/otadata + slot_a/b/c 布局 | 重写 |
| [sdkconfig.defaults](/sdkconfig.defaults) | factory-reset 引脚、按住时长、清 OTA 数据 | 扩展 |
| [main/CMakeLists.txt](/main/CMakeLists.txt) | 选择器源文件 + 槽A 合并目标 | 重写 |
| [tools/validate.sh](/tools/validate.sh) | 注册两项新宿主测试 | 扩展 |
| [assets/README.md](/assets/README.md) | 字体来源记录 | 扩展 |
| [README.md](/README.md) | 项目说明与用法 | 重写 |
| [docs/firmware-selector-tutorial.md](/docs/firmware-selector-tutorial.md) | 本教程 | 新增 |

未改动:`components/bsp/` 全部,以及基线硬件测试源码(`main/demo_*.c`、
`main/ui_pixel.*`——保留用于宿主测试与槽A 载荷构建,不参与选择器编译)。

## 3. 刷写选择器

依赖:数据 USB-C 线 + `esptool`(`pip install esptool`)。

```bash
# 全新安装(写入 bootloader + 分区表 + 选择器 + 槽A):
esptool --chip esp32c3 -p /dev/cu.usbmodem1101 -b 921600 \
    write_flash --verify 0x0 build/FoloToy-AI-Passport-full.bin
```

合并镜像只覆盖 `0x0..0x294600`,槽B/C 在刷入之前保持为空。

## 4. 往槽位里放固件

槽位只接受**纯 app 镜像**(以 `0xE9` 魔数开头),单个不超过 2MB:

| 槽位 | 偏移 | esptool 示例 |
| --- | --- | --- |
| A | `0x120000` | `write_flash 0x120000 你的固件.bin` |
| B | `0x320000` | `write_flash 0x320000 你的固件.bin` |
| C | `0x520000` | `write_flash 0x520000 你的固件.bin` |

玩法社区下载的是 `esp-merged-0x0` 完整镜像,不能直接进槽位,需要先提取出
app 镜像。推荐用配套工具 **extract-play**(图形界面或命令行):

- 图形界面 `extract_play_gui`:打开后**粘贴玩法链接**,或把合并镜像 `.bin`
  **拖进窗口**/点「选择本地文件…」,再点「提取 app 镜像」,即得到 `xxx-app.bin`。
- 命令行:`python extract_play.py "<玩法链接>"`(Windows 也可直接双击
  `extract_play.exe` 后粘贴链接)。

下载见[刷写助手仓库的 Releases](https://github.com/RainYufan/ai-passport-slot-flasher/releases)
(Windows `.exe`、macOS `.dmg`、Linux `.tar.gz`)。

没有工具时,也可用这段 Python 手动提取(注意**整段复制、不裁剪**,否则会切掉
镜像末尾的 SHA-256 校验摘要,导致刷入后无法启动):

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
    if kind == 0:  # app 分区
        open("extracted-app.bin", "wb").write(data[poff:])  # 整段,不裁剪
        break
    off += ENTRY.size
```

校验后刷入:

```bash
esptool image_info extracted-app.bin        # 应显示 Checksum/Validation Hash valid
esptool --chip esp32c3 -p /dev/cu.usbmodem1101 -b 921600 \
    write_flash --verify 0x470000 extracted-app.bin
```

选择器会读取每个镜像的应用描述(项目名、版本、日期),只列出有效固件。

## 5. 使用说明

| 操作 | 结果 |
| --- | --- |
| 开机 | 直进上次固件(首次开机进选择器菜单) |
| 开机按住任意按键约 2 秒 | 进入选择器菜单 |
| 上 / 下 | 移动选中项(会取消倒计时) |
| OK 短按 | 启动选中固件 |
| 10 秒无操作 | 自动启动上次固件 |

右上角显示电量(不可用时显示 `--`)。

## 6. 从源码构建

1. 安装 ESP-IDF **5.5.3**(ESP32-C3 目标),见
   [官方指南](https://docs.espressif.com/projects/esp-idf/en/v5.5.3/esp32c3/get-started/linux-macos-setup.html);
   PATH 里要有 `cmake` 与 `ninja`。
2. 激活并构建:

   ```bash
   . $HOME/esp/esp-idf-v5.5.3/export.sh
   idf.py set-target esp32c3
   idf.py build
   ```

3. 重建槽A 载荷与合并镜像:

   ```bash
   ./tools/build_hwtest_payload.sh
   ./tools/validate.sh --firmware   # 输出 build/FoloToy-AI-Passport-full.bin
   ```

4. 改了界面文字后,重新生成字体子集(lv_font_conv 1.5.3 + 有授权的 CJK 字
   体,如 Noto Sans SC)并跑字形门:

   ```bash
   ./tools/gen_fonts.sh
   ./tools/validate.sh --static
   ```

## 7. 自定义

- **槽位数量/大小**:改 [partitions.csv](/partitions.csv)(app 分区须 64KB 对齐
  且不超 8MB),同步修改 [main/boot_selector_model.h](/main/boot_selector_model.h)
  与 [tests/test_partition_layout.py](/tests/test_partition_layout.py) 里的常量。
- **进菜单按住时长**:[sdkconfig.defaults](/sdkconfig.defaults) 里的
  `CONFIG_BOOTLOADER_HOLD_TIME_GPIO`。
- **菜单文字**:改 `main/boot_ui.c` / `main/main.c` 的字符串字面量,然后跑
  [tools/gen_fonts.sh](/tools/gen_fonts.sh) 与静态门——出现字体未覆盖的字符
  时字形检查会直接失败。

## 8. 常见问题

**菜单里两个槽看起来一样。** 本仓库构建的很多应用项目名相同;槽卡片主行
是 `v<版本> · <日期>`,每次构建都唯一。

**按住开机进不了菜单。** 按住要满 2 秒;三个按键共用一条 ADC 线,
bootloader 层面任意按键都有效。

**槽位显示"空槽位"。** 要么没刷,要么镜像不是有效的 app 镜像;用
`esptool image_info` 复核。

**选了固件没启动。** bootloader 会校验镜像,校验失败自动回到选择器菜单;
重新刷写并校验该槽位。

**想找回原固件。** 本项目会替换原固件;原厂 RootPassport(约 3.52MB +
3MB voice 分区)放不进 2MB 槽位。可以刷任意 app 镜像固件进槽位。

## 9. 开源说明

- 许可:MIT(仓库 [LICENSE](/LICENSE));上游基线
  [FoloToy/ai-passport](https://gitee.com/FoloToy/ai-passport)
  ([GitHub 镜像](https://github.com/FoloToy/ai-passport))。
- 字体:Noto Sans SC 子集,SIL OFL 1.1,来源记录见
  [assets/README.md](/assets/README.md)。
- 预编译的 `payloads/hwtest-app.bin` 可由
  [tools/build_hwtest_payload.sh](/tools/build_hwtest_payload.sh) 从基线提交
  确定性重建。
- 本次交付验证状态:Build PASS、Host tests PASS、Device tests PASS(真机
  验证了启动、菜单、槽位识别与 OTA 分区切换),`actionlint` 未运行(离线环境)。
