# 协处理器 OTA（经 ESP-Hosted 传输链路）

> **适用摘要**: 在首次串口烧录协处理器固件后，后续的 slave 固件升级**复用现有的 ESP-Hosted 传输链路**（SDIO/SPI/UART）完成，无需额外硬件、ESP-Prog 或物理访问。host 通过 RPC 调用 `esp_hosted_slave_ota_begin/write/end/activate` 把固件分块写入协处理器并激活重启。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-hosted-mcu/resources/`, source/examples in `repos/esp-hosted-mcu/`, and this recipe path `repos/esp-hosted-mcu/recipes/slave_ota.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "远程升级 slave 固件"
- "ESP-Hosted slave OTA"
- "esp_hosted_slave_ota_begin"
- "协处理器 OTA 不接线"
- "host 给 slave 升级"

## 前置条件

| 条件 | 要求 |
|---|---|
| 链路 | 首次串口烧录已完成，ESP-Hosted 链路可正常 `connect_to_slave` |
| 固件来源 | OTA 镜像（web URL / host 分区 / 任意来源的字节流） |
| 头文件 | `#include "esp_hosted.h"`（含 `esp_hosted_ota.h`） |
| 参考例程 | `examples/host_performs_slave_ota/` |

## 分步说明

### 1. OTA 状态枚举（来自 `host/api/include/esp_hosted_ota.h`）

```c
enum {
    ESP_HOSTED_SLAVE_OTA_ACTIVATED,
    ESP_HOSTED_SLAVE_OTA_COMPLETED,
    ESP_HOSTED_SLAVE_OTA_NOT_REQUIRED,
    ESP_HOSTED_SLAVE_OTA_NOT_STARTED,
    ESP_HOSTED_SLAVE_OTA_IN_PROGRESS,
    ESP_HOSTED_SLAVE_OTA_FAILED,
};
```

### 2. 低层 OTA API（推荐用于新实现）

```c
esp_err_t esp_hosted_slave_ota_begin(void);
esp_err_t esp_hosted_slave_ota_write(uint8_t *ota_data, uint32_t ota_data_len);
esp_err_t esp_hosted_slave_ota_end(void);
esp_err_t esp_hosted_slave_ota_activate(void);   // 激活并重启协处理器
```

> 头文件中的 `esp_hosted_slave_ota(const char *image_url)` 已 **deprecated**，建议新实现直接用上面的 begin/write/end 三段式，自行取镜像字节流。底层 RPC 对应 `OTABegin`(272)/`OTAWrite`(273)/`OTAEnd`(274)。

### 3. 三段式 OTA 流程

```c
#include "esp_hosted.h"
#include "esp_log.h"

static const char *TAG = "slave_ota";

esp_err_t do_slave_ota(const uint8_t *image, size_t image_len)
{
    esp_err_t ret;

    // 0) 链路已就绪（esp_hosted_init + connect_to_slave 已调用）
    //    可选：先用 esp_hosted_get_coprocessor_fwversion 比对版本，避免无谓升级

    // 1) begin
    ret = esp_hosted_slave_ota_begin();
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "ota_begin failed: %s", esp_err_to_name(ret));
        return ret;
    }

    // 2) 分块 write（连续字节流，整包必须完整覆盖）
    const uint32_t chunk = 4096;   // 按例程/内存取舍
    for (size_t off = 0; off < image_len; off += chunk) {
        uint32_t len = (image_len - off < chunk) ? (image_len - off) : chunk;
        ret = esp_hosted_slave_ota_write((uint8_t *)(image + off), len);
        if (ret != ESP_OK) {
            ESP_LOGE(TAG, "ota_write @%u failed: %s", (unsigned)off, esp_err_to_name(ret));
            return ret;
        }
    }

    // 3) end
    ret = esp_hosted_slave_ota_end();
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "ota_end failed: %s", esp_err_to_name(ret));
        return ret;
    }

    // 4) activate（会重启协处理器；host 侧会收到 ESP_HOSTED_EVENT_CP_INIT）
    ret = esp_hosted_slave_ota_activate();
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "ota_activate failed: %s", esp_err_to_name(ret));
        return ret;
    }
    return ESP_OK;
}
```

### 4. 版本检查（避免无谓升级）

```c
esp_hosted_coprocessor_fwver_t ver;
if (esp_hosted_get_coprocessor_fwversion(&ver) == ESP_OK) {
    // 比对 ver 字段与目标镜像版本，决定是否跳过 OTA
}
```

更完整的应用描述符（含 magic word、secure version、IDF 版本、elf sha256 等）：

```c
esp_hosted_app_desc_t desc;
esp_hosted_get_coprocessor_app_desc(&desc);   // 部分字段需协处理器开启 full app descriptor
// desc.magic_word == ESP_HOSTED_APP_DESC_MAGIC_WORD (0xABCD5432)
```

### 5. 镜像来源（例程提供三种方式）

`examples/host_performs_slave_ota/` 演示了：① 从 web 服务器取镜像；② 从 host 分区取镜像；③ 版本检查跳过升级。具体目标芯片（ESP32-P4 / ESP32-H2）见该例程 README。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ota_write` 中途失败 | 传输不稳/校验失败 | 确保链路稳；启用 checksum；写入前确认 begin 成功 |
| 镜像不完整导致升级失败 | 字节流必须连续完整覆盖整包 | 不要跳块；按 off/len 顺序写完整 image_len |
| 激活后 slave 不起来 | 镜像版本/目标不匹配 | 核对镜像目标芯片与 slave 一致；必要时串口恢复 |
| 用了 deprecated 的 `esp_hosted_slave_ota(url)` | 旧 API | 改用 begin/write/end/activate 三段式 |
| 升级后版本未变 | 未 activate 或未重启 | `esp_hosted_slave_ota_activate()` 会触发重启 |

## 参考

- `host/api/include/esp_hosted_ota.h`
- `host/esp_hosted_misc.h`（`esp_hosted_app_desc_t`、`ESP_HOSTED_APP_DESC_MAGIC_WORD`）
- `examples/host_performs_slave_ota/README.md`（三种 OTA 方法与流程图）
- `docs/implemented_rpcs.md`（OTABegin/OTAWrite/OTAEnd RPC）
- `docs/migration_guide.md`（v2.6.0 引入 Slave OTA 的迁移说明）
