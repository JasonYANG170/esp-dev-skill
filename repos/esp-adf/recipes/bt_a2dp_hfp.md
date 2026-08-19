# 蓝牙服务：A2DP Sink / Source 与 HFP

> **适用摘要**: 用 `bluetooth_service` 创建蓝牙音频流元素与外设，接入 pipeline 实现蓝牙音箱（A2DP Sink）、蓝牙音源（A2DP Source）或免提通话（HFP）。`bluetooth_service_create_stream()` 返回可直接 link 的 audio element。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-adf/resources/`, source/examples in `repos/esp-adf/`, and this recipe path `repos/esp-adf/recipes/bt_a2dp_hfp.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "蓝牙音箱 / A2DP Sink"
- "蓝牙发音乐 / A2DP Source"
- "HFP 免提通话"
- "pipeline_a2dp_sink_stream 那个例子"
- "bluetooth_service"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/player/pipeline_a2dp_sink_stream/`、`examples/player/pipeline_a2dp_source_stream/`、`examples/player/pipeline_hfp_stream/`、`examples/get-started/pipeline_a2dp_sink_and_hfp/` |
| 头文件 | `components/bluetooth_service/include/bluetooth_service.h` |
| 蓝牙 | Classic BT（A2DP/HFP 需 ESP32 / ESP32-S3 / ESP32-C3 等支持 BT 的芯片；ESP32-S2 不支持 Classic BT） |

## 分步说明

### 1. 关键 API（来自 `bluetooth_service.h`）

```c
audio_element_handle_t bluetooth_service_create_stream(void);   // 创建蓝牙音频 element
esp_periph_handle_t    bluetooth_service_create_periph(void);   // 创建蓝牙外设
esp_err_t              bluetooth_service_start(bluetooth_service_cfg_t *config);  // 启动服务
```

### 2. A2DP Sink（手机推流 → 板子播放）拓扑

数据流：`[BT source(phone)] → bt_stream → i2s_stream(writer) → codec`

```c
#include "bluetooth_service.h"
#include "audio_pipeline.h"
#include "audio_element.h"
#include "i2s_stream.h"
#include "esp_peripherals.h"
#include "board.h"

// codec
audio_board_handle_t board = audio_board_init();
audio_hal_ctrl_codec(board->audio_hal, AUDIO_HAL_CODEC_MODE_DECODE, AUDIO_HAL_CTRL_START);

// pipeline
audio_pipeline_cfg_t pipeline_cfg = DEFAULT_AUDIO_PIPELINE_CONFIG();
audio_pipeline_handle_t pipeline = audio_pipeline_init(&pipeline_cfg);

// 蓝牙流元素
audio_element_handle_t bt_stream = bluetooth_service_create_stream();

// i2s 输出
i2s_stream_cfg_t i2s_cfg = I2S_STREAM_CFG_DEFAULT();
i2s_cfg.type = AUDIO_STREAM_WRITER;
audio_element_handle_t i2s_writer = i2s_stream_init(&i2s_cfg);

audio_pipeline_register(pipeline, bt_stream,  "bt");
audio_pipeline_register(pipeline, i2s_writer, "i2s");
const char *link_tag[2] = {"bt", "i2s"};
audio_pipeline_link(pipeline, link_tag, 2);
```

### 3. 启动 peripherals + 服务 + pipeline

```c
esp_periph_config_t periph_cfg = DEFAULT_ESP_PERIPH_SET_CONFIG();
esp_periph_set_handle_t set = esp_periph_set_init(&periph_cfg);

// 蓝牙外设（处理连接 / AVRCP 等事件）
esp_periph_handle_t bt_periph = bluetooth_service_create_periph();
esp_periph_start(set, bt_periph);

// 事件监听
audio_event_iface_cfg_t evt_cfg = AUDIO_EVENT_IFACE_DEFAULT_CFG();
audio_event_iface_handle_t evt = audio_event_iface_init(&evt_cfg);
audio_pipeline_set_listener(pipeline, evt);
audio_event_iface_set_listener(esp_periph_set_get_event_iface(set), evt);

audio_pipeline_run(pipeline);
// 主循环处理蓝牙连接状态 / 采样率变化事件
```

### 4. A2DP Source（板子发流 → 耳机/音箱）拓扑

数据流反过来：`[数据源(fatfs/http)] → decoder → bt_stream → [BT sink(耳机)]`。`bt_stream` 作为 writer。

### 5. HFP（免提通话）

HFP 同时涉及上行（mic→SCO→远端）和下行（远端→SCO→喇叭）两条路径，用专门的 HFP 配置。完整实现见 `examples/player/pipeline_hfp_stream/` 与 `examples/get-started/pipeline_a2dp_sink_and_hfp/`（同时跑 A2DP 和 HFP）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| Classic BT 不可用 | 用了 ESP32-S2（仅 BLE） | A2DP/HFP 改用 ESP32 / ESP32-S3 / ESP32-C3 |
| 连接后无声音 | codec 模式错或 i2s 方向错 | Sink 用 DECODE + i2s WRITER；AVRCP play 后才出声 |
| 采样率不对 | 未处理 BT 采样率事件 | 监听蓝牙事件，按需 `i2s_stream_set_clk` |
| 与 Wi-Fi 共存干扰 | 未开共存 | 参考 `examples/advanced_examples/wifi_bt_ble_coex/` |

## 参考项目

- `examples/player/pipeline_a2dp_sink_stream/` — 蓝牙音箱（Sink）
- `examples/player/pipeline_a2dp_source_stream/` — 蓝牙音源（Source）
- `examples/player/pipeline_bt_sink/`、`examples/player/pipeline_bt_source/` — 经典版
- `examples/player/pipeline_hfp_stream/` — HFP 免提
- `examples/get-started/pipeline_a2dp_sink_and_hfp/` — A2DP + HFP 并存
