# 新建 ESP-IDF 工程

> **适用摘要**: 从零创建一个可编译/烧录/监控的 ESP-IDF 工程，含目录结构、顶层与组件 CMakeLists、目标设置与构建流程。
> Excludes: Arduino-only sketches, ESP8266 RTOS SDK projects, and non-IDF component-only library work.
> ESP-IDF: Verify the user's installed version first; examples are based on standard ESP-IDF `hello_world` / `blink` layout.
> Validation: example-derived.

## 触发意图

- "新建 ESP-IDF 工程"
- "创建一个 ESP32 项目"
- "idf.py 怎么开始"
- "ESP-IDF 工程结构"
- "set-target / build / flash 流程"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考 | `examples/get-started/hello_world`、`examples/get-started/blink` |
| 环境 | 已安装 ESP-IDF 并激活（`export.bat` / `source export.sh`），`IDF_PATH` 已设 |
| 串口 | 开发板通过 USB 接入，已知端口名（COM3 / /dev/ttyUSB0 / /dev/cu.*） |

## 分步说明

### 1. 工程目录结构

```
my_project/
├── CMakeLists.txt          # 顶层固定三行
├── sdkconfig               # menuconfig 生成（勿手改）
├── main/
│   ├── CMakeLists.txt      # idf_component_register
│   └── main.c              # void app_main(void)
```

### 2. 顶层 CMakeLists.txt（顺序不可变）

```cmake
cmake_minimum_required(VERSION 3.22)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(my_project)
```

> 路径含空格不被支持；如需最小化构建可加 `idf_build_set_property(MINIMAL_BUILD ON)`（见 hello_world）。

### 3. main/CMakeLists.txt

```cmake
idf_component_register(SRCS "main.c"
                       INCLUDE_DIRS ".")
```

### 4. main/main.c（最小入口）

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"

static const char *TAG = "main";

void app_main(void)
{
    ESP_LOGI(TAG, "Hello ESP-IDF!");
    while (1) {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 5. 设置目标并构建

```bash
cd my_project
idf.py set-target esp32s3     # 或 esp32 / esp32c3 / esp32c6 / ...
idf.py build
```

### 6. 烧录并监控

```bash
idf.py -p COM3 flash monitor   # Windows
idf.py -p /dev/ttyUSB0 flash monitor   # Linux
```
- 退出监控：`Ctrl+]`
- 改分区表或 OTA 后整片擦除：`idf.py -p COM3 erase-flash flash`

### 7. 仅编译/烧录 app（省时）

```bash
idf.py app
idf.py -p COM3 app-flash
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `Target ... is not supported` | 未先 set-target | `idf.py set-target <chip>` |
| `undefined reference to ...` | 组件未在 `REQUIRES`/`PRIV_REQUIRES` | 在 `main/CMakeLists.txt` 加入对应组件名 |
| `Project path contains spaces` | 路径含空格 | 移到无空格目录 |
| 换芯片后编译错乱 | 旧构建残留 | `idf.py fullclean` 后重新 `set-target` |
| 烧录找不到端口 | 端口名错或驱动未装 | 确认设备管理器/`ls /dev/tty*`；装 USB-UART 驱动 |
| `idf.py` 命令未找到 | 环境未激活 | 每个新终端 `source export.sh` / 运行 `export.bat` |

## 参考

- `examples/get-started/hello_world` — 最小工程（含 `MINIMAL_BUILD`）
- `examples/get-started/blink` — GPIO/LED Strip 闪烁
- ESP-IDF `docs/en/api-guides/build-system.rst` — 构建系统详解
- ESP-IDF `README.md` — Quick Reference 命令清单
