# 自定义分区表 at_customize.csv

> **适用摘要**: 修改二级分区表 `at_customize.csv`，新增/调整用户数据分区，生成并烧录 `at_customize.bin`，为 `AT+SYSFLASH`、`AT+FS`、SSL 服务端、BLE 服务端等功能提供存储。

## 触发意图

- "自定义分区"
- "at_customize.csv"
- "增加用户分区"
- "AT+SYSFLASH 不可用"
- "AT+FS 报错"

## 前置条件

| 条件 | 要求 |
|---|---|
| 工程 | 已克隆 esp-at 并能编译（见 `recipes/build_and_flash.md`） |
| 模块 at_customize.csv | 位于 `module_config/module_<your_module>/at_customize.csv` |

## 分步说明

ESP-AT 有两张分区表：

- **`partitions_at.csv`**（系统主表）：基于其生成 `partitions_at.bin`。改错会导致系统无法启动，**不要修改**。
- **`at_customize.csv`**（二级表）：可自定义用户数据块。`AT+SYSFLASH`、`AT+FS`、SSL 服务端、BLE 服务端功能依赖它已烧录。

### 步骤 1：定位 at_customize.csv

按模块查找（来自 `How_to_customize_partitions.rst`）：

| 模块 | 路径 |
|---|---|
| ESP32 WROOM-32/PICO-D4/SOLO-1/MINI-1 | `module_config/module_esp32_default/at_customize.csv` |
| ESP32 WROVER-32 | `module_config/module_wrover-32/at_customize.csv` |
| ESP32-C3 MINI-1 | `module_config/module_esp32c3_default/at_customize.csv` |
| ESP32-C2 4MB | `module_config/module_esp32c2_default/at_customize.csv` |
| ESP32-C5 4MB | `module_config/module_esp32c5_default/at_customize.csv` |
| ESP32-C6 4MB | `module_config/module_esp32c6_default/at_customize.csv` |
| ESP32-C61 4MB | `module_config/module_esp32c61_default/at_customize.csv` |
| ESP32-S2 MINI | `module_config/module_esp32s2_default/at_customize.csv` |

### 步骤 2：按规则修改

修改规则（来自文档）：

- 已定义的用户分区，`Name` 和 `Type` 不能改，`SubType`/`Offset`/`Size` 可改。
- 新增分区：若 ESP-IDF `esp_partition.h` 已定义该 Type，沿用之；否则 Type 设 `0x40`。
- 用户分区 `Name` ≤ 16 字节。
- 新分区不能超出 `at_customize` 整体范围（范围由 `partitions_at.csv` 定义）。

示例：新增 4KB 分区 `test`：

```csv
# module_config/module_esp32c3_default/at_customize.csv
# Name,Type,SubType,Offset,Size
... ...
test,0x40,15,0x3E000,4K
fs_storage,data,0xff,0x47000,100K
```

ESP32/S2 示例（地址不同）：

```csv
test,0x40,15,0x3D000,4K
fs_storage,data,0xff,0x70000,576K
```

### 步骤 3：生成 at_customize.bin

方式一：重新编译工程（自动生成）。

```bash
./build.py build
```

方式二：用 ESP-IDF 脚本单独生成：

```bash
python esp-idf/components/partition_table/gen_esp32part.py \
    -q ./module_config/module_esp32c3_default/at_customize.csv at_customize.bin
```

### 步骤 4：烧录 at_customize.bin

烧录地址因模块而异（来自文档）：

| 芯片/模块 | 烧录地址 | 大小 |
|---|---|---|
| ESP32（多数）/ESP32-S2 | `0x20000` | `0xE0000` |
| ESP32-C2 2MB | `0x1A000` | `0x26000` |
| ESP32-C2 4MB | `0x1E000` | `0x42000` |
| ESP32-C3 / ESP32-C6 | `0x1E000` | `0x42000` |
| ESP32-C5 / ESP32-C61 | `0x30000` | `0x70000` |

```bash
esptool.py -p /dev/ttyUSB0 -b 460800 --chip auto \
    write_flash --flash_mode dio --flash_size detect --flash_freq 40m \
    0x1E000 ./at_customize.bin
```

> 用 `./build.py -p PORT flash` 会一并烧录所有分区，无需手动指定地址。

## 哪些功能强依赖 at_customize.bin

`at_customize.bin` 未烧录时以下指令不可用：

- `AT+SYSFLASH`（读写用户分区）
- `AT+FS`（文件系统）
- SSL 服务端相关指令
- BLE 服务端相关指令

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `AT+SYSFLASH` 报错 | at_customize.bin 未烧录 | 按地址烧录 at_customize.bin |
| 新分区超范围 | 超出 partitions_at.csv 定义的 at_customize 总大小 | 缩小分区或调整主表（谨慎） |
| Name 超长 | 超过 16 字节 | 缩短 Name |
| Type 与 ESP-IDF 冲突 | 自定义分区用了已占用 Type | 自定义分区 Type 用 `0x40` |
| 改了 partitions_at.csv | 改错表，可能无法启动 | 自定义只在 at_customize.csv 进行 |
| 生成报 CSV 格式错 | 地址/大小未对齐或单位写错 | Size 用 `4K`/`100K`，Offset 4K 对齐 |

## 参考

- 仓库文档：`docs/en/Compile_and_Develop/How_to_customize_partitions.rst`
- 分区示例：`module_config/module_esp32c3_default/at_customize.csv`
- 主分区表：`module_config/module_<name>/partitions_at.csv`
- 分区工具：`esp-idf/components/partition_table/gen_esp32part.py`
