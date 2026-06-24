---
name: esp-at-skill
description: >-
  AI Skill for Espressif ESP-AT AT-command firmware development. Used when users
  need to build, customize, or debug ESP-AT firmware, add user-defined AT commands,
  customize partitions/BLE services/AT port pins, perform OTA, and integrate
  Wi-Fi/BLE/TCP-IP/HTTP/MQTT/WebSocket functionality via AT commands on
  ESP32, ESP32-C2, ESP32-C3, ESP32-C5, ESP32-C6, ESP32-C61, and ESP32-S2.
  Trigger words: "ESP-AT", "AT command", "AT固件", "AT指令", "esp-at", "嘉立创 AT", "Espressif AT", "自定义AT命令", "AT+USEROTA", "AT through UART", "AT via SPI", "AT via SDIO"
tags:
  - embedded
  - esp-at
  - espressif
  - ESP32
  - ESP32-C3
  - ESP32-C6
  - AT-commands
  - wifi
  - bluetooth
  - firmware
  - OTA
license: Apache-2.0
compatibility: Build requires ESP-IDF (v5.4 for ESP32/C2/C3/C6/S2, v5.5 for C5/C61); targets ESP32, ESP32-C2, ESP32-C3, ESP32-C5, ESP32-C6, ESP32-C61, ESP32-S2 (ESP32-S3 and ESP32-H2 are NOT supported)
metadata:
  author: Community
  version: "1.0.0"
---

# esp-at-skill

面向 Espressif **ESP-AT** AT 指令固件的 AI 开发技能。ESP-AT 是基于 ESP-IDF 构建的官方 AT 固件平台，通过 UART/SPI/SDIO/socket 接口向外部 MCU 提供易于解析的 AT 指令集，快速集成 Wi-Fi、BLE、TCP-IP、HTTP、MQTT、WebSocket 等无线连接能力。本技能提供场景化配方（recipes）、真实 API 参考、配置选项速查与常见陷阱，全部内容均来自仓库真实文档与源码头文件。

## Core Principles

1. **永远不要猜测 API** — 所有函数、结构体、宏、Kconfig 选项必须能在 `components/at/include/` 或 `resources/` 中查到；查不到即视为不存在，绝不臆造。
2. **AT 核心库不开源** — 指令解析与设备操作实现在预编译库 `components/at/lib` 中，对外只暴露 `esp_at_core.h`/`esp_at.h` 头文件中的 API。自定义指令通过注册回调实现，不要试图改写核心库源码。
3. **自定义指令六步走** — 定义 `esp_at_cmd_t` 数组 → 实现 `esp_at_custom_cmd_register()` → 用 `ESP_AT_CMD_SET_INIT_FN()` 初始化 → 设置 `AT_CUSTOM_COMPONENTS` 环境变量 → 链接选项 `-u esp_at_custom_cmd_register` → 编译。任一步遗漏都会导致指令不生效。
4. **指令初始化顺序** — 内部指令集用 `ESP_AT_CMD_SET_FIRST_INIT_FN`，外部/自定义指令集用 `ESP_AT_CMD_SET_INIT_FN`（第二阶段）或 `ESP_AT_CMD_SET_LAST_INIT_FN`（最后阶段）。优先级数值 `p` 越大执行越晚。
5. **解析返回值三态** — `esp_at_get_para_as_digit/str` 返回 `ESP_AT_PARA_PARSE_RET_OK`/`_FAIL`/`_OMITTED`；处理可选参数时必须显式判断 `_OMITTED`，空串 `""` 不等于省略。
6. **命令名称规则** — 必须以 `+` 开头；允许字母、数字及 `! % - . / : _`；不支持的类型回调置 `NULL`。
7. **结果码语义** — `esp_at_write_result()` 仅输出结果串（OK/ERROR/SEND OK），不改接收任务状态；`esp_at_dispatch_result()` 输出结果串并把 AT 端口恢复到 ready 状态。指令完全结束、准备接收新数据时用后者。
8. **端口写数据有四变体** — `esp_at_port_write_data`（受 `AT+SYSMSGFILTER` 过滤、不唤醒 MCU）/ `_without_filter`（不过滤、不唤醒）/ `esp_at_port_active_write_data`（过滤、唤醒 MCU）/ `_without_filter`（不过滤、唤醒 MCU）。按是否需要消息过滤与是否唤醒 MCU 选择。
9. **模块决定配置** — `build.py install` 时选择的 Platform/Module 决定 `module_config/module_<name>/` 与 `factory_param_data.csv` 行，进而决定引脚、分区、Flash 大小等；改引脚应改 `factory_param_data.csv` 的 `uart_tx_pin/uart_rx_pin/uart_cts_pin/uart_rts_pin` 或用 `at.py modify_bin`。
10. **分区有两张表** — `partitions_at.csv`（系统主表，不要改）与 `at_customize.csv`（二级表，可自定义用户数据区）。`AT+SYSFLASH`、`AT+FS`、SSL 服务端、BLE 服务端功能依赖 `at_customize.bin` 已烧录。
11. **功能按需开启** — MQTT/HTTP/WebSocket/FS/Ethernet/Classic BT 等默认关闭（`CONFIG_AT_*_COMMAND_SUPPORT=n`），在 `menuconfig → AT` 或 `sdkconfig.defaults` 中开启；HTTPS 需要把 `AT_PROCESS_TASK_STACK_SIZE` 调到 4096 以上。
12. **三种 OTA 各有用途** — `AT+USEROTA`（自有 HTTP 服务器 URL，仅 app 区）、`AT+CIUPDATE`（iot.espressif.cn，可升级 app+用户分区）、`AT+WEBSERVER`（浏览器/微信小程序，仅 app 区）。

## When to Use

**Applicable:**
- 编译/烧录官方或自定义 ESP-AT 固件（本地 `build.py` 或 GitHub 网页编译）
- 添加用户自定义 AT 指令（`at_custom_cmd` 组件方式）
- 修改 AT 命令端口引脚、日志端口引脚
- 自定义分区表 `at_customize.csv`、增加用户数据分区
- 自定义 BLE GATT 服务（`gatts_data.csv`）
- 实现 OTA 升级（USEROTA/CIUPDATE/WEBSERVER）
- 通过 SPI/SDIO/socket 而非 UART 承载 AT 指令
- 启用/裁剪 Wi-Fi、BLE、TCP-IP、HTTP、MQTT、WebSocket、FS、Ethernet 功能
- 用 `at.py` 工具直接修改打包好的 `factory_XXX.bin` 配置

**Not applicable:**
- ESP32-S3、ESP32-H2（ESP-AT 明确不支持）
- 与 AT 无关的纯 ESP-IDF 应用开发（应使用 ESP-IDF 本身的技能/文档）
- PCB 硬件设计、原理图绘制
- 修改 AT 核心库内部行为（源码不开源，只能通过注册回调或 `__wrap_` 函数包裹拦截）

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先读取对应配方**——其中包含完整调用链、分步说明、真实代码与常见错误表。

### 工程构建与配置

| recipe | 场景 |
|---|---|
| `recipes/build_and_flash.md` | 本地克隆、安装环境、配置模块、编译、烧录 ESP-AT 工程 |
| `recipes/set_port_pin.md` | 修改 AT 命令端口/日志端口 UART 引脚（factory_param_data.csv 或 menuconfig） |
| `recipes/customize_partitions.md` | 自定义 `at_customize.csv` 用户分区、生成并烧录 `at_customize.bin` |
| `recipes/override_module_config.md` | 用 `at_override_module_config` 外部目录覆盖模块配置/补丁，不改 esp-at 源码 |

### 自定义 AT 指令与功能

| recipe | 场景 |
|---|---|
| `recipes/add_custom_command.md` | 添加用户自定义 AT 指令（四种类型、参数解析、结果输出、可选参数） |
| `recipes/customize_ble_service.md` | 自定义 BLE GATT 服务（修改 `gatts_data.csv`、perm/UUID 语义） |
| `recipes/ota_upgrade.md` | 实现 OTA 升级（USEROTA/CIUPDATE/WEBSERVER 三选一） |

### 接口与安全

| recipe | scenario |
|---|---|
| `recipes/at_over_spi_sdio.md` | 通过 SPI 或 SDIO（而非 UART）承载 AT 指令与主机 MCU 通信 |

---

## 芯片支持矩阵

ESP-AT 支持的 SoC（来自仓库 `README.md`，**ESP32-S3 与 ESP32-H2 不支持**）：

| 芯片 | master 分支 | 默认模块示例 | IDF 版本 |
|---|---|---|---|
| ESP32 | 支持 | WROOM-32, PICO-D4, SOLO-1, MINI-1, WROVER-32, ESP32-D2WD, ESP32-SDIO | v5.4 |
| ESP32-C2 | 支持 | ESP32C2-4MB, ESP32C2-2MB(-BLE/-G2/-NO-OTA-G2) | v5.4 |
| ESP32-C3 | 支持 | MINI-1, ESP32C3-SPI, ESP32C3_RAINMAKER | v5.4 |
| ESP32-C5 | 支持 | ESP32C5-4MB, ESP32C5-SDIO, ESP32C5-SPI | v5.5 |
| ESP32-C6 | 支持 | ESP32C6-4MB | v5.4 |
| ESP32-C61 | 支持 | ESP32C61-4MB | v5.5 |
| ESP32-S2 | 支持 | MINI | v5.4 |

---

## 模块默认 UART 引脚（factory_param_data.csv 摘录）

命令端口默认用 UART1，引脚在 `components/customized_partitions/raw_data/factory_param/factory_param_data.csv` 中按模块定义。`-1` 表示该模块不走 UART（如 SDIO/SPI 模块）。

| 模块 | uart_port | TX | RX | CTS | RTS |
|---|---|---|---|---|---|
| WROOM-32 | 1 | GPIO17 | GPIO16 | GPIO15 | GPIO14 |
| ESP32-C3 MINI-1 | 1 | GPIO7 | GPIO6 | GPIO5 | GPIO4 |
| ESP32C2-4MB | 1 | GPIO7 | GPIO6 | GPIO5 | GPIO4 |
| ESP32C5-4MB | 1 | GPIO23 | GPIO24 | GPIO25 | GPIO26 |
| ESP32C6-4MB | 1 | GPIO7 | GPIO6 | GPIO5 | GPIO4 |
| ESP32C61-4MB | 1 | GPIO7 | GPIO6 | GPIO5 | GPIO4 |
| ESP32-S2 MINI | 1 | GPIO17 | GPIO21 | GPIO20 | GPIO19 |
| ESP32-SDIO / ESP32C3-SPI / ESP32C5-SDIO / ESP32C5-SPI | -1 | -1 | -1 | -1 | -1 |

日志端口默认 UART0 引脚（来自 `How_to_set_AT_port_pin.rst`）：ESP32=GPIO1/3，ESP32-C2=GPIO20/19，ESP32-C3=GPIO21/20，ESP32-C5=GPIO11/12，ESP32-C6=GPIO16/17，ESP32-C61=GPIO11/10，ESP32-S2=GPIO43/44。

---

## 关键 Kconfig 速查（main/Kconfig）

| Kconfig 符号 | 默认 | 说明 |
|---|---|---|
| `AT_ENABLE` | y | 总开关 |
| `AT_BASE_ON_UART/SPI/SDIO/SOCKET` | UART | 通信方式（choice） |
| `AT_PROCESS_TASK_STACK_SIZE` | 2048 | AT 处理任务栈；HTTPS 需 ≥4096 |
| `AT_SOCKET_TASK_STACK_SIZE` | 6144（范围 2048–8196） | socket 任务栈 |
| `AT_SELF_COMMAND_SUPPORT` | n | 固件内部执行 AT 指令（`esp_at_exe_cmd`） |
| `AT_BASE_COMMAND_SUPPORT` | y | 基础指令集 |
| `AT_USER_COMMAND_SUPPORT` | y | 用户指令集 |
| `AT_USERWKMCU_COMMAND_SUPPORT` | y | `AT+USERWKMCU` 唤醒 MCU |
| `AT_WIFI_COMMAND_SUPPORT` | y | Wi-Fi 指令 |
| `AT_NET_COMMAND_SUPPORT` | y | TCP/IP 指令 |
| `AT_SOCKET_MAX_CONN_NUM` | 5 | 最大连接数 |
| `AT_MQTT_COMMAND_SUPPORT` | n | MQTT 指令 |
| `AT_HTTP_COMMAND_SUPPORT` | n | HTTP 指令（含 TX/RX buffer 配置） |
| `AT_WS_COMMAND_SUPPORT` | n | WebSocket 指令 |
| `AT_BLE_COMMAND_SUPPORT` | y（C2 为 n） | BLE 指令（依赖 BT_ENABLED） |
| `AT_BLUFI_COMMAND_SUPPORT` | y | BluFi 指令 |
| `AT_BT_COMMAND_SUPPORT` | n | Classic BT 指令（仅 ESP32） |
| `AT_FS_COMMAND_SUPPORT` | n | 文件系统指令 |
| `AT_FS_LITTLEFS/FATFS/FATFS_LEGACY` | LITTLEFS | FS 类型（分区标签 `fs_storage`，legacy 为 `fatfs`） |
| `AT_ETHERNET_SUPPORT` | n | Ethernet（仅 ESP32） |
| `AT_EAP_COMMAND_SUPPORT` | n | WPA2/WPA3-Enterprise |
| `AT_WEB_SERVER_SUPPORT` | n | Web Server（`AT+WEBSERVER`） |
| `AT_OTA_SUPPORT` | y | OTA 指令 |
| `AT_COMMAND_TERMINATOR_SUPPORT` | n | 自定义单字符指令终结符 |

---

## AT 指令处理回调签名（esp_at_core.h）

```c
// 单条指令描述符
typedef struct {
    char    *cmd_name;                          // 如 "+MYCMD"
    uint8_t (*test_cmd)(uint8_t *cmd_name);     // AT+MYCMD=?
    uint8_t (*query_cmd)(uint8_t *cmd_name);    // AT+MYCMD?
    uint8_t (*setup_cmd)(uint8_t para_num);     // AT+MYCMD=<...>
    uint8_t (*exe_cmd)(uint8_t *cmd_name);      // AT+MYCMD
} esp_at_cmd_t;

// 返回值（节选）
ESP_AT_RESULT_CODE_OK            // OK
ESP_AT_RESULT_CODE_ERROR         // ERROR
ESP_AT_RESULT_CODE_SEND_OK       // SEND OK
ESP_AT_RESULT_CODE_SEND_FAIL     // SEND FAIL
ESP_AT_RESULT_CODE_IGNORE        // 不输出

// 参数解析返回（注意是 _RET_ 不是 _RESULT_）
ESP_AT_PARA_PARSE_RET_FAIL / _OK / _OMITTED
```

---

## Critical Pitfalls (Must Read)

以下是最常见的错误。违反任何一条都会导致固件不工作或指令不生效。

### 1. 自定义指令未用 `ESP_AT_CMD_SET_INIT_FN` 初始化

```c
// ❌ WRONG — 只注册不初始化，链接器可能丢弃该函数，指令不生效
bool esp_at_custom_cmd_register(void) {
    return esp_at_custom_cmd_array_register(at_custom_cmd, sizeof(at_custom_cmd)/sizeof(esp_at_cmd_t));
}
// 缺少初始化宏

// ✅ CORRECT — 用宏强制放入 .at_cmd_set_init_fn 段，自动执行
ESP_AT_CMD_SET_INIT_FN(esp_at_custom_cmd_register, 1);
```

### 2. 未设置 `AT_CUSTOM_COMPONENTS` 环境变量

```bash
# ❌ WRONG — 直接编译，自定义组件未被纳入构建
./build.py build

# ✅ CORRECT — 先导出环境变量再编译
export AT_CUSTOM_COMPONENTS=/abs/path/to/at_custom_cmd   # Linux/macOS
set AT_CUSTOM_COMPONENTS=C:\abs\path\to\at_custom_cmd    # Windows
./build.py build
```

### 3. 解析返回值误用 `_RESULT_` 旧名

```c
// ❌ WRONG — 头文件里新枚举名是 _RET_，旧名 _RESULT_ 已 deprecated
if (esp_at_get_para_as_digit(0, &v) != ESP_AT_PARA_PARSE_RESULT_OK) { ... }

// ✅ CORRECT — 使用当前枚举名
if (esp_at_get_para_as_digit(0, &v) != ESP_AT_PARA_PARSE_RET_OK) { ... }
```

### 4. 可选参数未判断 `_OMITTED`

```c
// ❌ WRONG — 把省略当失败
esp_at_para_parse_ret_t r = esp_at_get_para_as_str(idx++, &s);
if (r != ESP_AT_PARA_PARSE_RET_OK) return ESP_AT_RESULT_CODE_ERROR;  // 省略时报错

// ✅ CORRECT — 省略与失败分开处理
esp_at_para_parse_ret_t r = esp_at_get_para_as_str(idx++, &s);
if (r == ESP_AT_PARA_PARSE_RET_FAIL) return ESP_AT_RESULT_CODE_ERROR;
if (r == ESP_AT_PARA_PARSE_RET_OMITTED) { /* 用默认值 */ }
```

### 5. 指令结束后未恢复 AT 端口就绪态

```c
// ❌ WRONG — 只 write_result，端口不恢复，下一条指令卡住
esp_at_port_write_data((uint8_t*)">", 1);
esp_at_write_result(ESP_AT_RESULT_CODE_OK);

// ✅ CORRECT — 需要恢复接收时用 dispatch_result
esp_at_port_write_data((uint8_t*)">", 1);
esp_at_dispatch_result(ESP_AT_RESULT_CODE_OK, NULL);
```

### 6. HTTPS 任务栈过小导致崩溃

```
# ❌ WRONG — 开启 AT_HTTP_COMMAND_SUPPORT 后仍用默认 2048 栈
CONFIG_AT_HTTP_COMMAND_SUPPORT=y
# AT_PROCESS_TASK_STACK_SIZE 保持默认 2048 → 建立 HTTPS 链路时栈溢出崩溃

# ✅ CORRECT — 升到 4096 以上
CONFIG_AT_HTTP_COMMAND_SUPPORT=y
CONFIG_AT_PROCESS_TASK_STACK_SIZE=4096
```

### 7. 改引脚改错了文件

```c
// ❌ WRONG — 改 partitions_at.csv 或在代码里硬编码引脚
// partitions_at.csv 是系统主表，改动可能导致无法启动

// ✅ CORRECT — 改 factory_param_data.csv 对应模块行的 uart_tx_pin/uart_rx_pin
// platform,module_name,...,uart_port,...,uart_tx_pin,uart_rx_pin,uart_cts_pin,uart_rts_pin,...
// PLATFORM_ESP32C3,MINI-1,...,1,...,7,6,5,4,1
// 或用 at.py modify_bin 直接改打包好的 factory bin
```

### 8. 用户分区 Name 过长或 Type 错误

```csv
# ❌ WRONG — Name 超过 16 字节；新增自定义类型用了 ESP-IDF 已占用 Type
my_very_long_partition_name,0x40,15,0x3E000,4K

# ✅ CORRECT — Name ≤16 字节；自定义分区 Type 用 0x40，已存在的类型沿用 ESP-IDF 定义
test,0x40,15,0x3E000,4K
fs_storage,data,0xff,0x47000,100K
```

### 9. 未烧录 `at_customize.bin` 导致 SSL/BLE/FS/`AT+SYSFLASH` 失败

```
# ❌ WRONG — 只烧 factory/ota bin，未烧 at_customize.bin
# → AT+SYSFLASH、AT+FS、SSL 服务端、BLE 服务端均不可用

# ✅ CORRECT — at_customize.bin 必须烧到二级分区表地址
# 例如 ESP32-C3: 0x1E000, ESP32-C5/C61: 0x30000, ESP32/S2: 0x20000, C2/C6: 0x1E000
esptool.py write_flash 0x1E000 at_customize.bin
```

### 10. OTA 后丢失 `AT+CIUPDATE` 能力

```
# ❌ WRONG — 初始版本用官方 token，刷入非官方固件后再也用不了 AT+CIUPDATE

# ✅ CORRECT — 若计划后续用 AT+CIUPDATE 升级自定义固件，
# 初始版本就应把 OTA token 配成自有 token（iot.espressif.cn 建设备取 key）
```

### 11. BLE 服务 perm 位含义混淆

```c
// ❌ WRONG — 把 perm 当成读权限位
// gatts_data.csv 第 2 行 0x2803 的 perm 是"特征属性"字节，不是访问权限

// ✅ CORRECT — 第二行(0x2803)的 perm 是特征属性位：
//   bit0 BROADCAST, bit1 READ, bit2 WRITE_NO_RESP, bit3 WRITE,
//   bit4 NOTIFY, bit5 INDICATE, bit6 SIGNED_WRITE, bit7 EXT_PROP
// 访问权限(ESP_GATT_PERM_READ 等)才是 0x01/0x02/0x10/0x20...
```

### 12. `esp_at_exe_cmd` 在指令回调中直接调用

```c
// ❌ WRONG — 在 test/query/setup/exe handler 内部直接调，会死锁
static uint8_t at_exe_cmd_test(uint8_t *cmd_name) {
    esp_at_exe_cmd("AT+CWJAP?", "OK", 3000);  // 禁止
    return ESP_AT_RESULT_CODE_OK;
}

// ✅ CORRECT — 通过独立任务+队列派发，回调里只投递
// CONFIG_AT_SELF_COMMAND_SUPPORT 需开启；该 API 仅用于初始化阶段预设指令
```

---

## Execution Workflow

| 步骤 | 名称 | 说明 |
|------|------|------|
| 1 | Plan | 明确需求：目标芯片/模块、所需 AT 功能集、是否自定义指令、接口（UART/SPI/SDIO/socket） |
| 2 | Recipe | 匹配 `recipes/` 中的场景，按其调用链与分步说明执行 |
| 3 | Query | 配方未覆盖的 API/配置，查 `resources/api_reference.md` 与 `resources/config_reference.md` |
| 4 | Validate | 核对所有函数签名（esp_at_core.h / esp_at.h）、Kconfig 符号、引脚、分区名 |
| 5 | Confirm | 向用户呈现方案：模块选择、引脚、分区、Kconfig、自定义组件路径 |
| 6 | Execute | 本地：`build.py install → menuconfig → build → -p PORT flash`；或 GitHub 网页编译 |
| 7 | Check | 验证：自定义指令已初始化、`AT_CUSTOM_COMPONENTS` 已设、分区已烧、栈足够 |
| 8 | Verify | 串口发 `AT`/`AT+GMR`/自定义指令验证；OTA 用 `AT+USEROTA`；BLE 用手机查服务 |

### 步骤 6 详情 — 模块/构建选择策略

**全新工程（首次构建）：**

1. `git clone --recursive https://github.com/espressif/esp-at.git`
2. `./build.py install` → 选 `Platform name`（如 `PLATFORM_ESP32C3`）→ 选 `Module name`（如 `MINI-1`）→ 选 silence mode（一般 No）
3. `./build.py menuconfig` → 在 `AT` 菜单按需开启功能
4. 自定义指令：把 `examples/at_custom_cmd/` 复制到工程外，`export AT_CUSTOM_COMPONENTS=...`
5. `./build.py build` → `build/factory/` 生成 `factory_XXX.bin`
6. `./build.py -p PORT flash`（若报 `ota data partition invalid`，先 `./build.py erase_flash`）

**已发布固件微调（不改源码）：** 用 `tools/at.py modify_bin` 直接改 `factory_XXX.bin` 的 Wi-Fi/证书/UART/GATTS 配置后烧录。

---

## Failure Strategies

| 情况 | 处理 |
|---|---|
| API 在 resources/ 查不到 | 停止，告知用户该 API 不存在；不要臆造 |
| 不确定通信方式 | 默认 UART；SDIO/SPI/socket 需对应模块（如 ESP32-SDIO、ESP32C3-SPI） |
| 自定义指令不生效 | 检查 `ESP_AT_CMD_SET_INIT_FN` 是否加、`AT_CUSTOM_COMPONENTS` 是否设、链接选项 `-u esp_at_custom_cmd_register` 是否加 |
| 引脚改后启动失败 | 确认改的是 `factory_param_data.csv` 不是 `partitions_at.csv`；命令引脚与日志引脚不能冲突 |
| SSL/BLE/FS 指令报错 | 确认 `at_customize.bin` 已烧到正确地址 |
| HTTPS 崩溃 | `AT_PROCESS_TASK_STACK_SIZE` 调到 ≥4096 |
| OTA 后变砖/丢能力 | 保留 OTA 备份分区；自定义固件初始版本用自有 OTA token |
| 不确定芯片是否支持 | 查芯片支持矩阵；ESP32-S3/H2 明确不支持 |

## References

- 场景配方 → `recipes/` 目录
- AT 公开 API 速查 → `resources/api_reference.md`
- Kconfig 配置选项速查 → `resources/config_reference.md`
- 常见陷阱汇总 → `resources/pitfalls.md`
- 仓库 examples 索引 → `resources/example_list.md`
- 仓库头文件（权威来源）→ `components/at/include/esp_at_core.h`、`esp_at.h`、`esp_at_cmd_register.h`、`esp_at_init.h`、`esp_at_types.h`
- 工厂参数表 → `components/customized_partitions/raw_data/factory_param/factory_param_data.csv`
- Kconfig → `main/Kconfig`
- 入口 → `main/app_main.c`
