# 构建与烧录（make / cmake 双轨）

> **适用摘要**: 用 make 或 cmake/`idf.py` 配置（menuconfig）、构建、烧录、擦除、监视 ESP8266_RTOS_SDK 项目。

## 触发意图

- "怎么编译"
- "make menuconfig"
- "烧录固件"
- "擦除 flash"
- "idf.py build"

## 前置条件

| 条件 | 要求 |
|---|---|
| 环境 | `IDF_PATH` 已设置；xtensa-lx106-elf 工具链在 PATH |
| 项目 | 已有含 `Makefile` / `CMakeLists.txt` 与 `main/` 的项目目录 |

## 分步说明

### make 路线（仓库示例原生）

```bash
export IDF_PATH=/path/to/ESP8266_RTOS_SDK
export PATH=$PATH:/path/to/xtensa-lx106-elf/bin

cd myProject
make menuconfig          # 配置：串口、Flash size、Partition table、Component config、Example config
make -j5                 # 构建 app + bootloader + partition table
make partition_table     # 仅生成分区表二进制并打印摘要
make flash               # 烧录 bootloader + partition + app（自动重建）
make app-flash           # 只烧 app（已烧过 bootloader/init data 时省时）
make erase_flash         # 整片擦除
make erase_flash flash   # 全擦后重烧
make monitor             # 串口监视（Ctrl-] 退出）
make flash monitor       # 烧完直接监视
make -j5 app-flash monitor   # 串行组合：并行编译 app 后烧录并监视
```

### cmake / idf.py 路线

```bash
idf.py menuconfig
idf.py build
idf.py -p /dev/ttyUSB0 flash
idf.py -p /dev/ttyUSB0 monitor
idf.py -p /dev/ttyUSB0 erase-flash
```

### 关键 menuconfig 节点

| 节点 | 作用 |
|---|---|
| `Serial flasher config` > `Default serial port` | 烧录串口 |
| `Serial flasher config` > `Flash size` | flash 大小（2MB/4MB/...），决定默认分区映射 |
| `Partition Table` | `Single factory app` / `Two OTA app` / `Custom partition table CSV` |
| `Component config` > `ESP8266-specific` > `PHY` | `vdd33_const`（影响 ADC 模式：255=测系统电压，否则测外部 TOUT） |
| `Component config` > `Log output` > `Default log verbosity` | 日志等级 |
| `<Example Configuration>` | 各示例自定义项（SSID、密码、服务器 IP 等，由 `Kconfig.projbuild` 提供） |

### 仅构建 app 与 app-flash

首次必须 `make flash`（含 bootloader + init data）。之后日常迭代用：

```bash
make -j5 app             # 仅编译 app
make app-flash           # 仅烧 app
```

### 并行编译

```bash
make -jN                 # N = CPU 核数 + 1
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `A fatal error occurred: Failed to connect to ESP8266` | 串口/波特率/进下载模式 | menuconfig 改串口；按 FLASH 上电；检查 USB-TTL |
| `make: *** No rule to make target` | 未在项目根目录执行 | `cd` 到含 `Makefile` 的项目根 |
| 编译极慢 | 未并行 | 加 `-j5` |
| `idf.py: command not found` | 未 source 环境 | `export` IDF_PATH 与 PATH，或用 make 路线 |
| 烧录后无输出 | 串口波特率不匹配 | app 用 115200；boot ROM 用 74880 |
| 改了 sdkconfig 不生效 | 缓存 | `make clean` 或删 `build/` 后重建 |

## 参考

- 仓库 `README.md` — Compiling / Flashing / Parallel Builds / Erasing Flash 章节
- `docs/en/api-guides/build-system.rst` — 构建系统、component、Makefile 说明
- `docs/en/api-guides/partition-tables.rst` — 分区表与 `make partition_table` / `make partition_table-flash`
