# AGENTS.md — Supplementary Agent Guide

> 核心规则、场景速查、陷阱、recipes 索引与执行流程都在 `SKILL.md`。
> 本文件只补充 `SKILL.md` 未覆盖的约定与工具链说明，**不重复**其内容。

## Project Context

**语言**: C / C++ · **目标**: ESP32-C6（HP Core + LP Core 非对称双核）· **框架**: esp-lowcode-matter
**产品代码位置**: 仅 LP Core（`products/<name>/main/*.cpp`）· **HP Core**: 跑 `pre_built_binaries/` 预编译镜像
**Toolchain**: ESP-IDF release/v5.3 + ESP-AMP（main）；构建 `idf.py`；烧录 `esptool.py`；可选 VS Code/Codespaces 扩展（"Lowcode:" 命令）

## Code Generation Conventions

### File Naming
- 应用入口/驱动：`app_main.cpp`、`app_driver.cpp`、`app_priv.h`（固定名，复制模板即有）
- 组件头文件：`<component>.h`（如 `button_driver.h`、`light_driver.h`、`temperature_sensor_sht30.h`）
- 构建脚本：每个目录一个 `CMakeLists.txt`
- 产品配置：`configuration/product_info.json`、`product_config.json`、`data_model_wifi.zap` / `data_model_thread.zap`、`cd_cert_*.der`

### Include Pattern（`app_main.cpp` / `app_driver.cpp`）
```cpp
// 必备
#include <stdio.h>
#include <system.h>
#include <low_code.h>
#include "app_priv.h"

// 按需（驱动/组件）
#include <button_driver.h>
#include <relay_driver.h>
#include <light_driver.h>
#include <temperature_sensor_sht30.h>
#include <occupancy_sensor_ld2420.h>
#include <display_ssd1306.h>
#include <i2c_master.h>      // I2C 初始化（SHT30/SSD1306）
#include <color_format.h>    // 灯光颜色结构（随 light_driver.h 带入）
```

### Standard Product Structure（来自 `docs/create_product.md`）
```
esp-lowcode-matter/
    products/
        <product_name>/
            configuration/
                data_model_wifi.zap        // Matter 数据模型（Wi-Fi）
                data_model_thread.zap      // Matter 数据模型（Thread）
                product_config.json        // 测试模式等附加配置
                product_info.json          // vendor/product/chip/connection 等
                cd_cert_fff1_8000.der      // CD 证书
            main/
                app_main.cpp               // setup()/loop()/main()/feature_update_from_system
                app_driver.cpp             // app_driver_init / 驱动 / app_driver_event_handler
                app_priv.h                 // 驱动函数与回调声明
                CMakeLists.txt             // idf_component_register(... REQUIRES ...)
            CMakeLists.txt                 // 顶层
            sdkconfig.defaults            // 如 CONFIG_BUTTON_DRIVER_USE_HP_GPIO=y
            README.md
    components/                            // 组件（button/relay/light/system/...）
    drivers/                               // 外设驱动（i2c/rmt/uart）
    pre_built_binaries/                    // HP Core 预编译镜像 + flash_args
    tools/mfg/                             // 证书/QR 码生成（mfg_low_code.sh）
```

### Canonical Entry / Main Pattern（`app_main.cpp`）
```cpp
#include <stdio.h>
#include <system.h>
#include <low_code.h>
#include "app_priv.h"

static const char *TAG = "app_main";

static void setup() {
    /* 1. 先注册回调 */
    low_code_register_callbacks(feature_update_from_system, event_from_system);
    /* 2. 再初始化驱动 */
    app_driver_init();
}

static void loop() {
    /* 拉取系统消息，触发对应回调 */
    low_code_get_feature_update_from_system();
    low_code_get_event_from_system();
}

int feature_update_from_system(low_code_feature_data_t *data) {
    /* 按 endpoint_id + feature_id 分发 */
    return app_driver_feature_update();   // 或具体 set_* 函数
}

int event_from_system(low_code_event_t *event) {
    return app_driver_event_handler(event);
}

extern "C" int main() {
    printf("%s: Starting low code\n", TAG);
    system_setup();   // 永远第一
    setup();
    while (1) {
        system_loop();
        loop();
    }
    return 0;
}
```

### Event Handler 模板（`app_driver_event_handler`，switch 覆盖 `low_code_event_type_t`）
```cpp
int app_driver_event_handler(low_code_event_t *event) {
    printf("%s: Received event: %d\n", TAG, event->event_type);
    switch (event->event_type) {
        case LOW_CODE_EVENT_SETUP_MODE_START: /* 启动指示灯效 */ break;
        case LOW_CODE_EVENT_SETUP_MODE_END:   /* 停止灯效 */ break;
        case LOW_CODE_EVENT_SETUP_SUCCESSFUL: /* 显示/日志 */ break;
        case LOW_CODE_EVENT_SETUP_FAILED:     /* 显示/日志 */ break;
        case LOW_CODE_EVENT_NETWORK_CONNECTED:
        case LOW_CODE_EVENT_NETWORK_DISCONNECTED:
        case LOW_CODE_EVENT_OTA_STARTED:
        case LOW_CODE_EVENT_OTA_STOPPED:
        case LOW_CODE_EVENT_READY:
        case LOW_CODE_EVENT_IDENTIFICATION_START:
        case LOW_CODE_EVENT_IDENTIFICATION_STOP:
        case LOW_CODE_EVENT_TEST_MODE_LOW_CODE:
        case LOW_CODE_EVENT_TEST_MODE_COMMON:
        case LOW_CODE_EVENT_TEST_MODE_BLE:
        case LOW_CODE_EVENT_TEST_MODE_SNIFFER:
        default: break;
    }
    return 0;
}
```

### Debug / Logging 约定
- LP Core 自带 `printf`，直接输出到 console（单线程同步，顺序可靠）。
- 用 `static const char *TAG = "app_main";`（或 `app_driver`），日志写 `printf("%s: ...\n", TAG, ...)`。
- 不要用动态日志级别库；定位 panic 用 `riscv32-esp-elf-addr2line -e <elf> <MEPC>`。

## Build Workflow（本地终端，见 `docs/getting_started_terminal.md`）

```sh
# 1. 环境（各仓库 export 对应路径）
git clone -b release/v5.3 https://github.com/espressif/esp-idf.git --recursive
git clone -b main https://github.com/espressif/esp-amp.git --recursive
git clone -b main https://github.com/espressif/esp-lowcode-matter.git --recursive
cd esp-lowcode-matter && ./install.sh && . ./export.sh

# 2. 选产品
export SELECTED_PRODUCT=light_cw_pwm
cd $LOW_CODE_PATH/products/$SELECTED_PRODUCT

# 3. Prepare Device（仅一次）：烧预编译 HP 镜像
cd $LOW_CODE_PATH/pre_built_binaries
esptool.py erase_flash
esptool.py write_flash $(cat flash_args)

# 4. Upload Configuration（仅一次）：生成证书 + QR + data_model.bin
cd $LOW_CODE_PATH/tools/mfg
export MAC_ADDRESS=<0123456789ABCDEF>
./mfg_low_code.sh $LOW_CODE_PATH/products/$SELECTED_PRODUCT esp32c6 $MAC_ADDRESS
esptool.py write_flash 0xD000 .../output/$MAC_ADDRESS/${MAC_ADDRESS}_esp_secure_cert.bin \
                       0x1F2000 .../output/$MAC_ADDRESS/${MAC_ADDRESS}_fctry.bin

# 5. Upload Code：构建 + 烧 LP 固件
cd $LOW_CODE_PATH/products/$SELECTED_PRODUCT
idf.py set-target esp32c6
idf.py build
esptool.py write_flash 0x20C000 build/$SELECTED_PRODUCT.bin

# 6. Console
python3 -m esp_idf_monitor
```

> Codespaces / VS Code 扩展对应按钮：Select Product → Select Chip → Select Port → Prepare Device → Upload Configuration → Upload Code。

## LowCode App Code Generation Checklist

- [ ] `main()` 中 `system_setup()` 是第一条应用语句
- [ ] `setup()` 内：先 `low_code_register_callbacks(feature_update_from_system, event_from_system)`，再 `app_driver_init()`
- [ ] 主循环 `while(1){ system_loop(); loop(); }`，`loop()` 内调用两个 `low_code_get_*_from_system()`
- [ ] `feature_update_from_system()` 按 `data->details.endpoint_id` + `data->details.feature_id` 分发
- [ ] 上报特性：`value.type` 与数据类型匹配；无 `feature_id` 时填 `low_level.matter.{cluster_id,attribute_id}`
- [ ] `app_driver_event_handler()` switch 覆盖所需 `LOW_CODE_EVENT_*`，并给出指示（灯效/显示）
- [ ] 周期上报用 `system_timer_create(cb, arg, timeout_ms, periodic)` + `system_timer_start()`；回调签名 `(system_timer_handle_t, void*)`
- [ ] 灯光：WS2812 用 `ws2812_io`，LED(PWM) 用 `led_io`（union 不可混用）
- [ ] 量程换算：Matter 亮度 0-255 → driver 0-100；色温 mireds ↔ Kelvin
- [ ] 无 `malloc/calloc`、无深递归；缓冲显式定边界
- [ ] `main/CMakeLists.txt` 的 `REQUIRES` 声明所有用到的组件
- [ ] 改 `data_model.zap` / `product_info.json` 后重跑「Upload Configuration」

## Do Not Modify

- `pre_built_binaries/` — HP Core 预编译镜像，只烧不改
- `components/`、`drivers/`、`tools/` — 框架源码与工具（新增能力放自己 product 或新组件目录）
- `SKILL.md` front matter — Skill 元数据
- 原仓库 `docs/` — 只读参考，引用而非改写
