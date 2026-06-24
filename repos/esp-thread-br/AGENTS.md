# AGENTS.md — Supplementary Agent Guide

> 核心规则、场景索引、陷阱、执行工作流均在 `SKILL.md`。本文件只补充 `SKILL.md` 未覆盖的工程约定与工具用法，不重复内容。

## Project Context

- **语言**: C (ESP-IDF 风格) · **目标**: ESP32 系列主控 SoC + ESP32-H2/C6 RCP 双芯片 Thread Border Router · **协议栈**: OpenThread (Thread 1.4 Certified)
- **工具链/构建**: ESP-IDF `idf.py`（>=5.1.0，仓库 README 推荐 `v5.5.4`）；CMake + 组件管理器 (`idf_component.yml`)
- **定位**: esp-thread-br 在 ESP-IDF + OpenThread 之上提供产品级 BR 扩展；线程网络层 API 来自 OpenThread，BR/RCP/OTA/Web 来自本仓库

## 代码生成约定

### 文件命名
- 应用入口：`main/esp_ot_br.c`（`app_main`）、`main/esp_ot_config.h`（radio/host/port + rcp_update 宏）
- 启动逻辑封装：`examples/common/thread_border_router/src/border_router_launch.c` + `include/border_router_launch.h`
- M5Stack 专用：`examples/common/thread_border_router_m5stack/`
- 组件头：`components/<comp>/include/<header>.h`

### Include 模式（典型 BR 工程）
```c
#include "esp_openthread.h"
#include "esp_openthread_border_router.h"
#include "esp_openthread_netif_glue.h"
#include "esp_openthread_types.h"
#include "esp_ot_config.h"             // 含 ESP_OPENTHREAD_DEFAULT_*_CONFIG()
#include "esp_rcp_update.h"            // esp_rcp_update_config_t, esp_rcp_update_init
#include "esp_ot_cli_extension.h"      // esp_cli_custom_command_init, OT_EXT_CLI_TAG
#include "esp_ot_ota_commands.h"       // esp_set_ota_server_cert
#include "esp_ot_wifi_cmd.h"           // esp_ot_wifi_connect 等
#include "esp_br_web.h"                // esp_br_web_start
#include "border_router_launch.h"      // launch_openthread_border_router
#include "openthread/instance.h"
#include "openthread/dataset_ftd.h"
#include "openthread/thread_ftd.h"
```

### 标准工程结构（basic_thread_border_router）
```
basic_thread_border_router/
├── main/
│   ├── esp_ot_br.c            # app_main: init → start → backbone → auto_start
│   ├── esp_ot_config.h        # ESP_OPENTHREAD_DEFAULT_RADIO/HOST/PORT_CONFIG + RCP_UPDATE 宏
│   ├── idf_component.yml      # 依赖 esp_ot_cli_extension / esp_rcp_update / esp_br_http_ota / esp_ot_br_server ...
│   └── CMakeLists.txt
├── server_certs/
│   └── ca_cert.pem            # OTA HTTPS 信任证书（自建服务器需替换）
├── partitions.csv             # nvs / otadata / ota_0 / ota_1 / web_storage / rcp_fw
├── sdkconfig.defaults         # 主配置（ESP32-S3, BR_DEV_KIT）
├── sdkconfig.defaults.esp32c5 # C5 standalone 覆盖
├── sdkconfig.defaults.esp32p4 # P4 覆盖
└── README*.md
examples/common/thread_border_router/   # 被复用组件：Kconfig.projbuild + border_router_launch
```

### 标准 `app_main` 模式
```c
void app_main(void) {
    size_t max_eventfd = 3;
#if CONFIG_OPENTHREAD_RADIO_SPINEL_SPI
    max_eventfd++;
#endif
#if CONFIG_OPENTHREAD_RADIO_TREL
    max_eventfd++;
#endif
    esp_vfs_eventfd_config_t eventfd_config = { .max_fds = max_eventfd };

    esp_openthread_config_t openthread_config = {
        .netif_config = ESP_NETIF_DEFAULT_OPENTHREAD(),
        .platform_config = {
            .radio_config = ESP_OPENTHREAD_DEFAULT_RADIO_CONFIG(),
            .host_config  = ESP_OPENTHREAD_DEFAULT_HOST_CONFIG(),
            .port_config  = ESP_OPENTHREAD_DEFAULT_PORT_CONFIG(),
        },
    };
    esp_rcp_update_config_t rcp_update_config = ESP_OPENTHREAD_RCP_UPDATE_CONFIG();

    ESP_ERROR_CHECK(esp_vfs_eventfd_register(&eventfd_config));
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(init_spiffs());                       // 挂 rcp_fw / web_storage
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    ESP_ERROR_CHECK(mdns_init());
    ESP_ERROR_CHECK(mdns_hostname_set("esp-ot-br"));
#if CONFIG_OPENTHREAD_CLI_OTA
    esp_set_ota_server_cert((char *)server_cert_pem_start);
#endif
#if CONFIG_OPENTHREAD_BR_START_WEB
    esp_br_web_start("/spiffs");
#endif

    launch_openthread_border_router(&openthread_config, &rcp_update_config);
}
```

`launch_openthread_border_router()` 内部：`ot_console_start` → (可选)`ot_external_coexist_init` → (AUTO_UPDATE_RCP) `esp_rcp_update_init` + `esp_ot_register_rcp_handler` → `esp_openthread_start` → `esp_ot_update_rcp_if_different` → `esp_cli_custom_command_init` + `ot_register_external_commands` → (AUTO_START) `xTaskCreate(ot_br_init, ...)`。

## 构建工作流

1. 拉取并初始化 ESP-IDF（`./install.sh` + `. ./export.sh`），切到推荐 tag（如 `v5.5.4`）。
2. 构建 RCP：`cd $IDF_PATH/examples/openthread/ot_rcp && idf.py set-target esp32h2 && idf.py build`（默认 UART 460800）。
3. 进入 BR 工程：`cd esp-thread-br/examples/basic_thread_border_router`。
4. 选目标：`idf.py set-target <esp32s3|esp32c5|esp32p4|...>`。
5. `idf.py menuconfig` 调板型 / 引脚 / `OPENTHREAD_BR_*`。
6. `idf.py build`（构建期自动把 ot_rcp 打包进 `rcp_fw` SPIFFS，并按 `CREATE_OTA_IMAGE_WITH_RCP_FW` 生成 `build/ota_with_rcp_image`）。
7. `idf.py -p PORT flash monitor`。
8. 验证日志：`RCP reset: RESET_POWER_ON` → `RCP API Version: N` → `OpenThread attached to netif` → `Role detached -> leader`。

## 代码生成 Checklist

- [ ] `app_main` 中 eventfd 数量按 SPI/TREL 累加
- [ ] `nvs_flash_init` / `esp_netif_init` / `esp_event_loop_create_default` 三件套已调用
- [ ] `init_spiffs()` 在 `CONFIG_AUTO_UPDATE_RCP` 时注册 `rcp_fw` 分区，`OPENTHREAD_BR_START_WEB` 时注册 `web_storage`
- [ ] `esp_ot_config.h` 的 radio/host/port 宏与板型引脚一致；SPI 模式两端同步
- [ ] `ESP_OPENTHREAD_RCP_UPDATE_CONFIG()` 与 `ESP_BR_RCP_TARGET_ID`（H2/C6）匹配
- [ ] `OPENTHREAD_BR_AUTO_START` 时 `ot_br_init` 中先 `esp_openthread_set_backbone_netif(get_example_netif())` 再 `esp_openthread_border_router_init()`
- [ ] dataset 缺失时用 `otDatasetCreateNewNetwork` 生成，再 `esp_openthread_auto_start(&dataset)`
- [ ] `sdkconfig.defaults` 含全部 lwIP hook 与 `LWIP_IPV6_NUM_ADDRESSES=12`（IDF v5.3.1+）
- [ ] OTA 自建服务器时 `server_certs/ca_cert.pem` 已替换并 `fullclean`
- [ ] partitions.csv 含 `rcp_fw`/`web_storage`/双 ota 分区
- [ ] `idf_component.yml` 依赖版本：`esp_ot_cli_extension ~2.0.0`、`esp_rcp_update ~1.6.0`

## Do Not Modify
- `components/`、`examples/common/` 源码（属仓库本体）；定制应复制后修改副本。
- `SKILL.md` front matter（Skill 元数据）。
- OpenThread 栈头文件（`openthread/*.h`）属 IDF/OT 上游。
