# 从 ESP-SR V1.* 迁移到 V2.0

> **适用摘要**: 把基于 ESP-SR V1.*（`AFE_CONFIG_DEFAULT`、`ESP_AFE_SR_HANDLE` 等）的旧代码迁移到 V2.0 的新 API（`afe_config_init` + `esp_afe_handle_from_config`）。

## 触发意图

- "迁移到 V2.0"
- "AFE_CONFIG_DEFAULT 编译不过"
- "ESP_AFE_SR_HANDLE 找不到"
- "升级 esp-sr 后报错"

## 前置条件

| 条件 | 要求 |
|---|---|
| 旧代码 | 使用 V1.* 接口（`AFE_CONFIG_DEFAULT`、`ESP_AFE_SR_HANDLE`、`ESP_AFE_VC_HANDLE`） |
| 目标版本 | ESP-SR >= 2.0（当前 `idf_component.yml` 为 `2.4.6`） |
| 参考文档 | `docs/en/audio_front_end/migration_guide.rst` |

## 分步说明

### 1. 输入数据格式改用字符串

V2.0 用 `input_format` 字符串定义通道排列，不再用结构体字段逐个设：

| 字符 | 含义 |
|---|---|
| `M` | 麦克风通道 |
| `R` | 播放参考通道（AEC 用） |
| `N` | 未知 / 未用通道 |

`MMNR` = 4 通道：麦、麦、未用、参考。**数据必须是通道交错（interleaved）排布。**

### 2. 替换配置初始化

```c
// ❌ V1.* — 已移除
afe_config_t *afe_config = AFE_CONFIG_DEFAULT("MMNR", models, AFE_INTERNAL);

// ✅ V2.0
afe_config_t *afe_config = afe_config_init(
    "MMNR", models, AFE_TYPE_SR, AFE_MODE_HIGH_PERF);
afe_config_print(afe_config);   // 可选：打印完整配置核对
```

`afe_config_init` 会根据芯片与 input_format 尽量开启所有算法，再手动微调字段（如 `cfg->aec_init`、`cfg->wakenet_model_name`）。

### 3. 替换 handle 获取

```c
// ❌ V1.* — 已移除
const esp_afe_sr_iface_t *afe_handle = ESP_AFE_SR_HANDLE;   // 语音识别
const esp_afe_sr_iface_t *afe_handle = ESP_AFE_VC_HANDLE;   // 语音通信

// ✅ V2.0 — handle 由配置决定（type/mode 内含在 afe_config 里）
const esp_afe_sr_iface_t *afe_handle = esp_afe_handle_from_config(afe_config);
```

原来区分 SR/VC 靠两个不同宏，现在靠 `afe_config->afe_type`（`AFE_TYPE_SR` / `AFE_TYPE_VC` / `AFE_TYPE_VC_8K` / `AFE_TYPE_FD`）。

### 4. 创建实例的 API 名字变了

```c
// ❌ V1.* — create()
esp_afe_sr_data_t *afe_data = afe_handle->create(afe_config);

// ✅ V2.0 — create_from_config()
esp_afe_sr_data_t *afe_data = afe_handle->create_from_config(afe_config);
afe_config_free(afe_config);   // 创建完即可释放配置
```

### 5. 完整迁移对照

| V1.* | V2.0 |
|---|---|
| `AFE_CONFIG_DEFAULT(fmt, models, alloc)` | `afe_config_init(fmt, models, AFE_TYPE_*, AFE_MODE_*)` |
| `ESP_AFE_SR_HANDLE` | `esp_afe_handle_from_config(cfg)`（`cfg->afe_type=AFE_TYPE_SR`） |
| `ESP_AFE_VC_HANDLE` | `esp_afe_handle_from_config(cfg)`（`cfg->afe_type=AFE_TYPE_VC`） |
| `afe_handle->create(cfg)` | `afe_handle->create_from_config(cfg)` |
| `AFE_INTERNAL` / `AFE_PSRAM` 内存标志 | `cfg->memory_alloc_mode`（`AFE_MEMORY_ALLOC_MORE_INTERNAL` / `_INTERNAL_PSRAM_BALANCE` / `_MORE_PSRAM`） |
| 通道靠字段描述 | `input_format` 字符串（`M`/`R`/`N`） |

### 6. 迁移后验证

```c
afe_config_t *cfg = afe_config_init("MR", models, AFE_TYPE_SR, AFE_MODE_HIGH_PERF);
const esp_afe_sr_iface_t *afe_handle = esp_afe_handle_from_config(cfg);
esp_afe_sr_data_t *afe_data = afe_handle->create_from_config(cfg);
afe_handle->print_pipeline(afe_data);   // 确认 pipeline 含 AEC/NS/VAD/WakeNet
afe_config_free(cfg);
```

`afe_config_check()` 还能在配置冲突时自动修正（如双麦时优先 BSS 而非 NS）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `undefined reference to AFE_CONFIG_DEFAULT` | V1.* 宏已删 | 换 `afe_config_init` |
| `undefined reference to ESP_AFE_SR_HANDLE` | V1.* 宏已删 | 换 `esp_afe_handle_from_config` |
| `create` 不在结构体里 | V2.0 改名 `create_from_config` | 全局替换 |
| 迁移后 AEC 没生效 | `afe_config_init` 默认按 input_format 开 AEC，但 `input_format` 没 `R` | 加上 `R` 通道；或 `cfg->aec_init=true` |
| VC 8k 输入错 | 用了 `AFE_TYPE_VC` 喂 8k 数据 | 8k 输入必须用 `AFE_TYPE_VC_8K` |

## 参考

- `docs/en/audio_front_end/migration_guide.rst` — 官方迁移指南
- `docs/en/audio_front_end/README.rst` — V2.0 AFE API
- `include/esp32s3/esp_afe_config.h` — `afe_config_init`、`afe_mode_t`、`afe_type_t`、`afe_config_check`
- 配套 recipe：`recipes/afe_sr_pipeline.md`
