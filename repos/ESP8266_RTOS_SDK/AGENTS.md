# AGENTS.md — 补充约定

> 核心规则、场景速查、陷阱、recipes 索引、执行工作流都在 `SKILL.md`。
> 本文件只写 `SKILL.md` 未覆盖的工程约定与工具链指引，不重复内容。

## Project Context

**Language**: C（仓库以 C 为主） · **Target**: Espressif ESP8266EX（Tensilica L106 32-bit 单核） · **Framework**: ESP8266_RTOS_SDK（esp-idf style，FreeRTOS） · **Toolchain**: `xtensa-lx106-elf-gcc` v8.4.0；构建系统 GNU Make（`make/project.mk`）与 CMake 并存。

## Code Generation Conventions

### File Naming
- 应用源文件：`main/*.c`，配套 `main/component.mk` / `main/CMakeLists.txt`
- 自定义 component：`components/<name>/{<name>.c, include/<name>.h, component.mk, CMakeLists.txt}`
- Kconfig 选项定义：`Kconfig.projbuild`（项目级菜单）或 `Kconfig`（component 内）
- 分区表：`partitions*.csv`（CSV），构建期由 `tools/gen_esp32part.py` 转二进制烧到 0x8000

### Include Pattern
```c
// FreeRTOS
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "freertos/queue.h"

// 系统
#include "esp_system.h"
#include "esp_log.h"
#include "esp_err.h"
#include "esp_event.h"

// WiFi（注意：本仓库同时存在 tcpip_adapter 与 esp_netif 两种风格）
#include "esp_wifi.h"
#include "tcpip_adapter.h"   // 旧风格（station/softap/espnow 示例）
// 或
#include "esp_netif.h"       // 新风格（http_request/simple_ota 示例）

// NVS
#include "nvs_flash.h"
#include "nvs.h"

// 外设驱动（头在 components/esp8266/include/driver/）
#include "driver/gpio.h"
#include "driver/uart.h"
#include "driver/i2c.h"
#include "driver/spi.h"
#include "driver/pwm.h"
#include "driver/adc.h"
#include "driver/hw_timer.h"
```

### Standard Project Structure
```
myProject/
├── CMakeLists.txt            # 顶层 cmake，include($ENV{IDF_PATH}/tools/cmake/project.cmake); project(myProject)
├── Makefile                  # PROJECT_NAME := myProject \n include $(IDF_PATH)/make/project.mk
├── sdkconfig                 # make menuconfig 生成
├── sdkconfig.defaults        # 默认配置（可选）
├── partitions_example.csv    # 自定义分区表（可选，menuconfig 指定）
├── main/
│   ├── CMakeLists.txt        # idf_component_register(SRCS "main.c" INCLUDE_DIRS ".")
│   ├── component.mk          # COMPONENT_ADD_INCLUDEDIRS / COMPONENT_SRCDIRS（make 用）
│   ├── Kconfig.projbuild     # 项目级菜单项（如 SSID/密码/服务器地址）
│   └── main.c                # 内含 app_main()
└── components/               # 可选自定义 component
    └── my_lib/
        ├── CMakeLists.txt
        ├── component.mk
        ├── include/my_lib.h
        └── my_lib.c
```

### Canonical Entry / Init Pattern（WiFi 项目）
```c
static const char *TAG = "app";

void app_main(void)
{
    // 1. NVS（WiFi 之前必做）
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    // 2. 网络栈 + 事件循环（二选一成套使用）
    // 旧：
    tcpip_adapter_init();
    // 新：
    // esp_netif_init();
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    // 3. WiFi
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
    // 注册事件 → set_mode → set_config → start → connect(在 STA_START 事件里)
    // ...详见 recipes/wifi_station.md

    // 4. 创建业务任务（不要在 app_main 里死循环）
    xTaskCreate(my_task, "my_task", 4096, NULL, 5, NULL);
}
```

### Task Creation
```c
// 栈单位为 byte（ESP8266 FreeRTOS），典型 2048~8192
xTaskCreate(my_task, "my_task", 4096, arg, 5, &handle);
// 延时
vTaskDelay(1000 / portTICK_PERIOD_MS);   // 等价 portTICK_RATE_MS
```

### Logging
```c
#include "esp_log.h"
static const char *TAG = "myapp";
ESP_LOGI(TAG, "value=%d free_heap=%u", v, esp_get_free_heap_size());
// 等级由 menuconfig: Component config → ESP32-specific → LOG_DEFAULT_LEVEL 控制
// 也支持 ESP_LOGE / ESP_LOGW / ESP_LOGD / ESP_LOGV
```

### Error Handling
```c
esp_err_t err = esp_wifi_start();
ESP_ERROR_CHECK(err);                 // 失败时 abort 并打印 backtrace
// 或
if (err != ESP_OK) {
    ESP_LOGE(TAG, "fail: %s", esp_err_to_name(err));
    return err;
}
```

## Build Workflow（make 为主）

```bash
# 0. 一次性环境
export IDF_PATH=/path/to/ESP8266_RTOS_SDK
export PATH=$PATH:/path/to/xtensa-lx106-elf/bin

# 1. 配置
cd myProject
make menuconfig          # 串口 / Flash size / Partition table / Component config / Example config

# 2. 构建（并行）
make -j5

# 3. 烧录（自动重建）
make flash               # 含 bootloader + partition + app
make app-flash           # 只烧 app
make erase_flash         # 全擦
make erase_flash flash   # 全擦后重烧

# 4. 监视
make monitor             # Ctrl-] 退出
make flash monitor       # 烧完直接监视
```

### CMake / idf.py（仓库已支持）
```bash
idf.py menuconfig
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```
> 示例目录同时有 `Makefile` 与 `CMakeLists.txt`；两条路线任选其一，`sdkconfig` 共享。

## ESP8266_RTOS_SDK Code Generation Checklist

- [ ] 入口为 `app_main()`，内部不写死循环（长逻辑用 `xTaskCreate`）
- [ ] WiFi 前调用 `nvs_flash_init()`（失败先 `nvs_flash_erase` 再 init）
- [ ] 网络栈与事件循环二选一成套：`tcpip_adapter_init()` 或 `esp_netif_init()` + `esp_event_loop_create_default()`
- [ ] WiFi 初始化顺序：init → 注册事件 → `set_mode` → `set_config` → `start` → (STA) 在 `STA_START` 事件里 `connect`
- [ ] STA 在 `WIFI_EVENT_STA_DISCONNECTED` 里带计数重连
- [ ] 应用等 `IP_EVENT_STA_GOT_IP` 再发网络请求
- [ ] 所有初始化类返回 `esp_err_t` 的 API 用 `ESP_ERROR_CHECK`
- [ ] GPIO 不使用 GPIO6~GPIO11；GPIO16 不上拉（仅下拉）
- [ ] UART 先 `uart_param_config` 再 `uart_driver_install` 才能 read/write
- [ ] PWM 修改 period/duty/phase/invert 后必须 `pwm_start()`
- [ ] OTA 写完镜像后 `esp_ota_set_boot_partition` + `esp_restart`
- [ ] SPIFFS / 自定义分区：menuconfig 启用 Custom partition CSV 且 CSV 含对应分区
- [ ] ADC 批量读前关 WiFi 与中断
- [ ] 日志统一用 `ESP_LOGx` + `TAG`，不用 printf 做正式日志

## Do Not Modify

- `IDF_PATH` 指向的 SDK 源码树（`components/`、`make/`、`tools/`）—— 只读
- `sdkconfig` 由 `make menuconfig` 生成，手改需谨慎（建议用 `sdkconfig.defaults` + menuconfig）
- 本技能的 `resources/` 是文档来源，不要改其内容
- `SKILL.md` 的 frontmatter（技能元数据）
