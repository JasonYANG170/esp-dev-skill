# 配置参考（Kconfig / menuconfig）

> 所有配置项取自仓库真实 `Kconfig*` 文件。在示例目录用 `idf.py menuconfig` 修改。

## ESP Agent Config

来源：`components/agent/Kconfig.projbuild`

| 配置项 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_AGENT_API_ENDPOINT` | string | `api.agents.espressif.com` | ESP Private Agents API endpoint；自建部署改为自己的 URL |

宏引用：`esp_agent_core.h` 中 `#define ESP_AGENT_API_ENDPOINT CONFIG_ESP_AGENT_API_ENDPOINT`。

## ESP Agent Setup Configuration

来源：`components/setup/Kconfig.projbuild`

| 配置项 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_AGENT_SETUP_RAINMAKER_DEVICE_NAME` | string | `Espressif AI Agent` | RainMaker App 中显示的设备名 |
| `CONFIG_AGENT_SETUP_DEFAULT_AGENT_ID` | string | `""` | 默认 agent ID；为空时需运行时 `set-agent` 或 App 下发 |
| `CONFIG_AGENT_SETUP_CREATE_RAINMAKER_DEVICE` | bool | `y` | 是否创建 RainMaker 设备用于配置 agent/volume；关闭则不可经 App 改 agent ID（user auth service 始终创建） |

## App Common Config

来源：`examples/common/app_common/Kconfig`

| 配置项 | 类型 | 默认 | 范围 | 说明 |
|---|---|---|---|---|
| `CONFIG_APP_EMOTE_PARTITION_LABEL` | string | `anim_icon` | — | 表情资源分区标签 |
| `CONFIG_APP_EMOTE_TASK_CORE_ID` | int | 0 | 0–1 | 绑定 gfx emote 任务的核心 |
| `CONFIG_APP_AUDIO_DEFAULT_PLAYBACK_VOLUME` | int | 75 | 0–100 | 默认播放音量 |
| `CONFIG_AUDIO_UPLOAD_SAMPLE_RATE` | int | 8000 | — | 上行采样率 Hz |
| `CONFIG_AUDIO_DOWNLOAD_SAMPLE_RATE` | int | 16000 | — | 下行采样率 Hz |
| `CONFIG_AUDIO_UPLOAD_FRAME_DURATION_MS` | int | 20 | — | 上行帧时长 ms |
| `CONFIG_AUDIO_DOWNLOAD_FRAME_DURATION_MS` | int | 60 | — | 下行帧时长 ms |

## Audio Config

来源：`components/audio/Kconfig`

| 配置项 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ENABLE_AEC` | bool | `n` | 启用 AEC 回声消除（须板硬件支持） |

## ESP Client Only Matter Controller Config

来源：`examples/matter_controller/components/matter_controller/Kconfig`

| 配置项 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_CLIENT_ONLY_MATTER_CONTROLLER` | bool | `n` | 启用纯客户端 Matter 控制器 |

依赖与互斥（来自 help）：
- `depends on ESP_MATTER_CONTROLLER_ENABLE`
- `depends on !ESP_MATTER_ENABLE_MATTER_SERVER`
- `depends on !ESP_MATTER_COMMISSIONER_ENABLE`
- `depends on !WIFI_NETWORK_COMMISSIONING_DRIVER`
- `depends on !THREAD_NETWORK_COMMISSIONING_DRIVER`
- `depends on !ENABLE_CHIPOBLE`
- `default n`
- `select ENABLE_CHIP_CONTROLLER_BUILD`
- `select RMAKER_REST_API_ENABLED`

> 即开启此项前必须先关闭 Matter server / commissioner / Wi-Fi & Thread commissioning driver / CHIPOBLE。

## 板级宏（board_defs.h，非 Kconfig）

来源：`examples/common/boards/<board>/board_defs.h`。

| 宏 | esp_box_3 | esp_vocat_board_v1_2 | 说明 |
|---|---|---|---|
| `LCD_MIRROR_X_Y` | 1 | — | LCD 水平+垂直镜像 |
| `LEDC_BACKLIGHT_SUPPORTED` | 1 | 1 | 支持 LEDC 背光 |
| `CAPACITIVE_TOUCH_SUPPORTED` | — | 1 | 支持电容触摸 |
| `CAPACITIVE_TOUCH_CHANNEL_GPIO` | — | 7 | 电容触摸通道 GPIO |
| `INDICATOR_DEVICE_NAME` | — | `"led_green"` | 监听状态指示 LED 设备名 |
| `BOARD_DEVICE_MANUAL_URL` | 已定义 | 已定义 | RainMaker App 内手册 URL |

> m5stack_cores3 与 m5stack_cores3_h2_gateway 的 board_defs.h 另行定义（结构相同）。自定义板请参考 `recipes/add_custom_board.md`。

## 组件依赖版本（idf_component.yml）

| 组件/示例 | 依赖 | 版本 |
|---|---|---|
| `components/agent` | `espressif/esp_websocket_client` | ^1.6.0（IDF ≥ 5.0） |
| `components/audio` | `espressif/esp_codec_dev` | ^1.5（public） |
| `components/audio` | `espressif/esp-sr` | ^2.1.5 |
| `components/audio` | `espressif/gmf_ai_audio` | ^0.7.2 |
| `components/audio` | `espressif/gmf_audio` | ^0.7.1 |
| `components/audio` | `espressif/esp_audio_simple_player` | ^0.9 |
| `components/audio` | `espressif/gmf_io` | ^0.7 |
| `examples/common/app_common` | `espressif2022/esp_emote_expression` | 0.0.* |
| `examples/common/app_common` | `espressif/network_provisioning` | * |
| `examples/*/main` | idf | ≥ 5.5 |
