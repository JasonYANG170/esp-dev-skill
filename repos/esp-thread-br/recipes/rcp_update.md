# RCP 更新机制

> **适用摘要**: 让主控 SoC 自动/手动更新 RCP（ESP32-H2/C6）固件；理解序列号、verified flag 与自动回滚。

## 触发意图
- "更新 RCP 固件"
- "otrcp update"
- "AUTO_UPDATE_RCP"
- "RCP 回滚"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_AUTO_UPDATE_RCP=y`（自动）、`CONFIG_OPENTHREAD_RCP_COMMAND=y`（手动 `otrcp`） |
| 分区 | `rcp_fw` SPIFFS 已挂载，存储路径 `/rcp_fw/ot_rcp_idx/` |
| 引脚 | 主控连 RCP 的 RESET 与 BOOT（ESP32-H2 BOOT=GPIO8） |
| 参考 | `docs/en/dev-guide/rcp_update.rst`、`components/esp_rcp_update/include/esp_rcp_update.h` |

## 分步说明

### 1. 初始化 RCP 更新

`app_main` 通过 `ESP_OPENTHREAD_RCP_UPDATE_CONFIG()` 宏填充配置（`esp_ot_config.h`）：

```c
#define ESP_OPENTHREAD_RCP_UPDATE_CONFIG()                                                                   \
    {                                                                                                        \
        .rcp_type = RCP_TYPE_UART, .uart_rx_pin = CONFIG_PIN_TO_RCP_TX, .uart_tx_pin = CONFIG_PIN_TO_RCP_RX, \
        .uart_port = 1, .uart_baudrate = 115200, .reset_pin = CONFIG_PIN_TO_RCP_RESET,                       \
        .boot_pin = CONFIG_PIN_TO_RCP_BOOT, .update_baudrate = 460800,                                       \
        .firmware_dir = "/" CONFIG_RCP_PARTITION_NAME "/ot_rcp", .target_chip = ESP_BR_RCP_TARGET_ID         \
    }
```

启动流程（`launch_openthread_border_router`）：
```c
#if CONFIG_AUTO_UPDATE_RCP
    ESP_ERROR_CHECK(esp_rcp_update_init(update_config));
    esp_ot_register_rcp_handler();
#endif
    ESP_ERROR_CHECK(esp_openthread_start(config));
#if CONFIG_AUTO_UPDATE_RCP
    esp_ot_update_rcp_if_different();   // 版本不一致则更新
#endif
```

### 2. RCP 镜像存储规则

- 镜像存于 SPIFFS `/rcp_fw/ot_rcp_idx/`，`idx` 由 `esp_rcp_get_update_seq()` 决定。
- **序列号与 verified flag**（NVS 中）：
  - verified=true → 当前镜像 idx = RCP sequence number
  - verified=false → 当前镜像 idx = 1 − RCP sequence number
- 默认值：sequence=0, verified=1。

### 3. 自动更新与回滚

启用 `AUTO_UPDATE_RCP` 后：
- BR 首发或检测到 RCP 启动失败 → 从 SPIFFS 烧写当前 idx 镜像到 RCP。
- OTA 下载新 RCP 镜像时，先把当前镜像标为备份（改 sequence + verified flag），下载完成后切换 idx；若新 RCP 启动失败则回滚到备份。

相关 API（`esp_rcp_update.h`）：

```c
esp_err_t esp_rcp_update_init(const esp_rcp_update_config_t *update_config);
esp_err_t esp_rcp_update(void);
const char *esp_rcp_get_firmware_dir(void);
int8_t esp_rcp_get_update_seq(void);
int8_t esp_rcp_get_next_update_seq(void);
void esp_rcp_reset(void);
esp_err_t esp_rcp_submit_new_image(void);
esp_err_t esp_rcp_mark_image_verified(bool verified);
esp_err_t esp_rcp_mark_image_unusable(void);
esp_err_t esp_rcp_load_version_in_storage(char *version_str, size_t size);
void esp_rcp_update_deinit(void);
```

### 4. 手动 `otrcp update`（不重启主控）

需 `CONFIG_OPENTHREAD_RCP_COMMAND=y`（命令 `esp_openthread_process_rcp_command`）。

```
> thread stop
I OPENTHREAD:[N] Mle-----------: Role leader -> detached -> disabled
> ifconfig down
I OT_STATE: netif u
> otrcp update
...
Done
> ifconfig up
> thread start
```

### 5. OTA 期间用 esp_rcp_ota_* 流式接收

`components/esp_rcp_update/include/esp_rcp_ota.h` 提供 RCP OTA 流式接口：

```c
typedef enum {
    ESP_RCP_OTA_STATE_READ_HEADER = 0,
    ESP_RCP_OTA_STATE_DOWNLOAD_RCP_FW,
    ESP_RCP_OTA_STATE_FINISHED,
    ESP_RCP_OTA_STATE_INVALID,
} esp_rcp_ota_state_t;

esp_err_t esp_rcp_ota_begin(esp_rcp_ota_handle_t *out_handle);
esp_rcp_ota_state_t esp_rcp_ota_get_state(esp_rcp_ota_handle_t handle);
uint32_t esp_rcp_ota_get_subfile_size(esp_rcp_ota_handle_t handle, esp_rcp_filetag_t filetag);
esp_err_t esp_rcp_ota_receive(esp_rcp_ota_handle_t handle, const void *data, size_t size, size_t *received_size);
esp_err_t esp_rcp_ota_end(esp_rcp_ota_handle_t handle);
esp_err_t esp_rcp_ota_abort(esp_rcp_ota_handle_t handle);
```

镜像 filetag（`esp_rcp_firmware.h`）：
```
FILETAG_RCP_VERSION=0, FILETAG_RCP_FLASH_ARGS=1, FILETAG_RCP_BOOTLOADER=2,
FILETAG_RCP_PARTITION_TABLE=3, FILETAG_RCP_FIRMWARE=4, FILETAG_HOST_FIRMWARE=5,
FILETAG_IMAGE_HEADER=0xff
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ESP_ERR_NOT_FOUND RCP firmware not found` | `rcp_fw` 分区无镜像 | 先构建 ot_rcp 让 `create_ota_image.py` 打包；确认 SPIFFS 已挂 |
| `otrcp` 命令不存在 | 未启用 | `CONFIG_OPENTHREAD_RCP_COMMAND=y` |
| 手动更新后 RCP 异常 | Thread 未停就更新 | 先 `thread stop` + `ifconfig down` |
| 升级后不断重启 | 新 RCP 启动失败，回滚触发中 | 等待自动回滚；或检查镜像 filetag 顺序 |
| `target_chip` 不匹配 | H2/C6 选错 | 改 `ESP_BR_H2_TARGET` / `ESP_BR_C6_TARGET` |

## 参考
- `docs/en/dev-guide/rcp_update.rst`（2.4）
- `docs/en/dev-guide/ota_update.rst`（2.3 镜像结构）
- `components/esp_rcp_update/include/esp_rcp_update.h`
- `components/esp_rcp_update/include/esp_rcp_ota.h`
- `components/esp_rcp_update/include/esp_rcp_firmware.h`
- `components/esp_ot_cli_extension/include/esp_ot_rcp_commands.h`
