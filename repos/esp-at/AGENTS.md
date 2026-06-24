# AGENTS.md — 补充代理指南

> 核心规则、配方索引、陷阱、工作流均在 `SKILL.md` 中。
> 本文件仅记录 `SKILL.md` 未覆盖的约定与工具说明，不重复内容。

## Project Context

**语言**: C · **目标**: Espressif ESP-AT AT 指令固件（基于 ESP-IDF） · **工具链**: ESP-IDF v5.4（ESP32/C2/C3/C6/S2）、v5.5（C5/C61） · **构建脚本**: `build.py`（封装 `idf.py`）

支持芯片：ESP32、ESP32-C2、ESP32-C3、ESP32-C5、ESP32-C6、ESP32-C61、ESP32-S2。**ESP32-S3 与 ESP32-H2 不支持。**

## 文件命名约定

- 自定义指令组件目录：`at_custom_cmd/`（可置于 esp-at 仓库外）
  - 源码：`at_custom_cmd/custom/*.c`、`at_custom_cmd/include/*.h`
  - 构建脚本：`at_custom_cmd/CMakeLists.txt`
- 模块覆盖目录：`at_override_module_config/`（外部覆盖，不改 esp-at 源码）
- 模块配置：`module_config/module_<name>/`（含 `sdkconfig.defaults`、`sdkconfig_silence.defaults`、`at_customize.csv`、`patch/`）
- 工厂参数：`components/customized_partitions/raw_data/factory_param/factory_param_data.csv`
- BLE 服务数据：`components/customized_partitions/raw_data/ble_data/gatts_data.csv`
- Web 页面：`components/fs_image/index.html`
- AT 工具：`tools/at.py`（修改打包好的 factory bin）

## Include 模式

自定义指令源文件只需包含：

```c
#include "esp_at.h"   // 拉入 esp_at_core.h、esp_at_cmd_register.h、esp_at_legacy.h、nvs.h
```

若需 Wi-Fi/TCP-IP/HTTP/WebSocket 等可选头，由对应 `CONFIG_AT_*_COMMAND_SUPPORT` 在 `esp_at_core.h` 内条件包含（如 `esp_wifi.h`、`sys/socket.h`、`esp_http_client.h`、`esp_websocket_client.h`），无需手动 `#include`。

## 标准自定义组件结构

```
at_custom_cmd/
├── CMakeLists.txt          # idf_component_register + WHOLE_ARCHIVE TRUE
├── README.md
├── include/
│   └── at_custom_cmd.h
└── custom/
    └── at_custom_cmd.c     # 实现 esp_at_custom_cmd_register + ESP_AT_CMD_SET_INIT_FN
```

### 标准 CMakeLists.txt 模板（来自 examples/at_custom_cmd）

```cmake
file(GLOB_RECURSE srcs *.c)
set(includes "include")
# 用到 at/freertos/nvs_flash 以外的组件时在此追加
set(require_components at freertos nvs_flash)

idf_component_register(
    SRCS ${srcs}
    INCLUDE_DIRS ${includes}
    REQUIRES ${require_components})

idf_component_set_property(${COMPONENT_NAME} WHOLE_ARCHIVE TRUE)
```

> `WHOLE_ARCHIVE TRUE` 是关键，确保 `ESP_AT_CMD_SET_INIT_FN` 放入的段不被链接器丢弃。

### 标准自定义指令骨架（来自 examples/at_custom_cmd/custom/at_custom_cmd.c）

```c
#include "esp_at.h"

static const esp_at_cmd_t at_custom_cmd[] = {
    {"+TEST", at_test_cmd_test, at_query_cmd_test, at_setup_cmd_test, at_exe_cmd_test},
};

bool esp_at_custom_cmd_register(void)
{
    return esp_at_custom_cmd_array_register(at_custom_cmd,
                                            sizeof(at_custom_cmd) / sizeof(esp_at_cmd_t));
}

ESP_AT_CMD_SET_INIT_FN(esp_at_custom_cmd_register, 1);
```

## 入口与初始化模式

`main/app_main.c` 是唯一入口（**不要修改**，它已固定调用 `esp_at_init()`）：

```c
void app_main(void)
{
    esp_at_main_preprocess();
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_at_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_at_init();   // 内部调用 esp_at_cmd_set_register()，自动执行所有 *_INIT_FN 段
}
```

自定义指令的注册通过 `ESP_AT_CMD_SET_INIT_FN` 宏被 `esp_at_init()` 自动调用，**无需**也**不能**在 `app_main` 里手动注册。

## 构建工作流

```bash
# 1. 克隆（含子模块）
git clone --recursive https://github.com/espressif/esp-at.git
cd esp-at

# 2. 安装环境（首次：选 Platform / Module / silence mode）
./build.py install            # Linux/macOS
python build.py install       # Windows

# 3. 配置（可选，按需开启功能）
./build.py menuconfig         # → AT 菜单

# 4. 设自定义组件环境变量（若有 at_custom_cmd / at_override_module_config）
export AT_CUSTOM_COMPONENTS="/abs/path1 /abs/path2"

# 5. 编译（输出 build/factory/factory_XXX.bin）
./build.py build

# 6. 烧录
./build.py -p /dev/ttyUSB0 flash
# 报 "ota data partition invalid" 时：./build.py erase_flash 后重烧

# 7. 高级：所有 idf.py 子命令都可用
./build.py --help
```

## 自定义指令代码生成检查清单

- [ ] 指令名以 `+` 开头，字符仅含 `A-Z a-z 0-9 ! % - . / : _`
- [ ] `esp_at_cmd_t` 数组中四种类型回调按需实现，不用的置 `NULL`
- [ ] 注册函数返回 `bool`，调用 `esp_at_custom_cmd_array_register()`
- [ ] 用 `ESP_AT_CMD_SET_INIT_FN(fn, priority)`（外部组件）或 `_FIRST_INIT_FN`（内部）/`_LAST_INIT_FN`
- [ ] 解析返回值用 `_RET_OK/_RET_FAIL/_RET_OMITTED`（非旧名 `_RESULT_`）
- [ ] 可选参数显式判断 `_OMITTED`；空串 `""` 不算省略
- [ ] 指令结束需恢复端口时用 `esp_at_dispatch_result()`，仅输出用 `esp_at_write_result()`
- [ ] 端口写数据按需选四变体（`_active_` 唤醒 MCU、`_without_filter` 旁路 SYSMSGFILTER）
- [ ] `CMakeLists.txt` 设 `WHOLE_ARCHIVE TRUE` 并追加所需 `require_components`
- [ ] 设 `AT_CUSTOM_COMPONENTS` 环境变量指向组件目录

## 函数包裹（function wrapping）约定

需要拦截/增强 AT 核心行为时，用链接器 `__wrap_` 包裹（不改核心库源码）。仓库示例 `examples/at_storage_security` 即用此法加密 NVS：

```cmake
# 在组件 CMakeLists.txt 中用 --wrap 链接选项
target_link_libraries(${COMPONENT_LIB} INTERFACE "-Wl,--wrap=nvs_set_blob")
```

```c
// 实现包裹函数，原函数用 __real_ 前缀调用
esp_err_t __wrap_nvs_set_blob(nvs_handle_t h, const char *key, const void *v, size_t len) {
    /* 加密后调用真实实现 */
    return __real_nvs_set_blob(h, key, encrypted, enc_len);
}
```

## Do Not Modify

- `components/at/lib/` — 预编译 AT 核心库，不开源
- `components/at/include/` — 公开头文件（API 契约）
- `main/app_main.c` — 固定入口
- `partitions_at.csv`（系统主分区表，改了可能导致无法启动）
- `SKILL.md` 的 frontmatter（技能元数据）

如需定制模块配置/补丁，使用 `at_override_module_config` 外部目录覆盖，不要直接改 esp-at 仓库源码。
