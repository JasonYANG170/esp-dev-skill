# 第一个项目 hello_world

> **适用摘要**: 从零创建第一个 ESP8266_RTOS_SDK（esp-idf style）项目，完成配置、构建、烧录与串口监视，验证工具链与环境就绪。

## 触发意图

- "新建 ESP8266 项目"
- "hello world 示例"
- "怎么开始 ESP8266_RTOS_SDK"
- "第一个固件"

## 前置条件

| 条件 | 要求 |
|---|---|
| 工具链 | xtensa-lx106-elf-gcc v8.4.0 已安装并加入 PATH |
| SDK | 已 `git clone` ESP8266_RTOS_SDK，`IDF_PATH` 指向其根目录 |
| Python 依赖 | `pip install -r $IDF_PATH/requirements.txt` |
| 硬件 | ESP8266 开发板（如 NodeMCU、ESP-12F）经 USB 串口连接 PC |
| 参考示例 | `examples/get-started/hello_world/` |

## 分步说明

### 1. 复制 hello_world 示例

```bash
cp -r $IDF_PATH/examples/get-started/hello_world ~/my_hello
cd ~/my_hello
```

### 2. 设置环境

```bash
export IDF_PATH=/path/to/ESP8266_RTOS_SDK
export PATH=$PATH:/path/to/xtensa-lx106-elf/bin
```

### 3. 配置（首次会弹出菜单）

```bash
make menuconfig
```
进入 `Serial flasher config` > `Default serial port`，填入串口（Linux `/dev/ttyUSB0`、macOS `/dev/cu.SLAB_USBtoUART`、Windows `COM3`）。设置 `Flash size`（如 4MB / `4MB`）。保存退出，生成 `sdkconfig`。

### 4. 构建

```bash
make -j5
```

### 5. 烧录 + 监视

```bash
make flash monitor
```
监视器里看到循环打印与倒计时，最后 `Restarting now.` 并复位。退出监视器按 `Ctrl-]`。

### 6. 关键源码（来自示例）

```c
// examples/get-started/hello_world/main/hello_world_main.c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_system.h"
#include "esp_spi_flash.h"

void app_main()
{
    printf("Hello world!\n");

    esp_chip_info_t chip_info;
    esp_chip_info(&chip_info);
    printf("This is ESP8266 chip with %d CPU cores, WiFi, ", chip_info.cores);
    printf("silicon revision %d, ", chip_info.revision);
    printf("%dMB %s flash\n", spi_flash_get_chip_size() / (1024 * 1024),
            (chip_info.features & CHIP_FEATURE_EMB_FLASH) ? "embedded" : "external");

    for (int i = 10; i >= 0; i--) {
        printf("Restarting in %d seconds...\n", i);
        vTaskDelay(1000 / portTICK_PERIOD_MS);
    }
    printf("Restarting now.\n");
    fflush(stdout);
    esp_restart();
}
```

> 注意：`app_main()` 由系统主任务调用，返回后该任务被删除（这里因为最后 `esp_restart()` 不返回）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `make: xtensa-lx106-elf-gcc: Command not found` | 工具链未加 PATH | 下载 v8.4.0 工具链，解压后 `export PATH` |
| `IDF_PATH not set` | 未导出 IDF_PATH | `export IDF_PATH=/path/to/ESP8266_RTOS_SDK` |
| 烧录失败 `Failed to connect` | 串口选错 / 板子未进下载模式 | menuconfig 改串口；按住 FLASH 键上电 |
| 串口乱码 | 波特率不对 | boot ROM 日志为 74880；app `printf` 走 UART0 默认 115200 |
| `nvs_flash_init` / `esp_wifi` 报错 | 与本例无关，但若改加 WiFi 需先 init NVS | 见 `recipes/wifi_station.md` |

## 参考

- `examples/get-started/hello_world/` — 官方入门示例
- `examples/README.md` — 示例分类与使用说明
- 仓库 `README.md` — Getting Started（toolchain / IDF_PATH / menuconfig / make flash）
