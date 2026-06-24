# ESP-DL 配置参考（Kconfig + ESP-PPQ 量化参数）

> Kconfig 符号取自 `esp-dl/Kconfig`；ESP-PPQ 参数取自 `docs/en/tutorials/` 与示例脚本。所有名称均为真实符号。

## 一、ESP-IDF Kconfig（`ESP-DL Configuration` 菜单）

> 仅视觉/图像像素转换支持开关。默认全部开启；为省 Flash/代码体积可按需关闭不用的转换。

### Pixel Convert Support（`PIX_CVT_*`）

| Kconfig 符号 | 说明 | 默认 |
|---|---|---|
| `PIX_CVT_RGB565_TO_RGB565_SUPPORT` | rgb565 → rgb565 | y |
| `PIX_CVT_RGB565_TO_RGB888_SUPPORT` | rgb565 → rgb888 | y |
| `PIX_CVT_RGB565_TO_GRAY_SUPPORT` | rgb565 → gray | y |
| `PIX_CVT_RGB565_TO_HSV_SUPPORT` | rgb565 → hsv | y |
| `PIX_CVT_RGB888_TO_RGB888_SUPPORT` | rgb888 → rgb888 | y |
| `PIX_CVT_RGB888_TO_RGB565_SUPPORT` | rgb888 → rgb565 | y |
| `PIX_CVT_RGB888_TO_GRAY_SUPPORT` | rgb888 → gray | y |
| `PIX_CVT_RGB888_TO_HSV_SUPPORT` | rgb888 → hsv | y |
| `PIX_CVT_GRAY_TO_GRAY_SUPPORT` | gray → gray | y |
| `PIX_CVT_HSV_TO_HSV_MASK_SUPPORT` | hsv → hsv_mask | y |
| `PIX_CVT_YUV_TO_RGB888_SUPPORT` | yuv → rgb888 | y |
| `PIX_CVT_YUV_TO_RGB565_SUPPORT` | yuv → rgb565 | y |
| `PIX_CVT_YUV_TO_GRAY_SUPPORT` | yuv → gray | y |
| `PIX_CVT_YUV_TO_HSV_SUPPORT` | yuv → hsv | y |
| `PIX_CVT_YUV_TO_YUV_SUPPORT` | yuv → yuv | y |

配置方式：`idf.py menuconfig` → `ESP-DL Configuration` → `Vision/Image: Pixel Convert Support`。

## 二、组件清单关键字段（`idf_component.yml`，来自 `esp-dl/idf_component.yml`）

| 字段 | 真实值（v3.3.6） |
|---|---|
| `version` | `"3.3.6"` |
| `license` | `"MIT"` |
| `targets` | esp32, esp32c2, esp32c3, esp32c5, esp32c6, esp32p4, esp32s2, esp32s3, esp32s31 |
| `idf` 版本约束 | 多数 target `>=5.3`；esp32c5 `>=5.5`；esp32s31 `>=6.0` |
| `dependencies` | `espressif/esp_new_jpeg: "^1"`、`espressif/dl_fft: ">=0.4.0"` |

## 三、`dl::Model` 构造参数（影响内存/性能的关键开关）

| 参数 | 类型 | 默认 | 作用 |
|---|---|---|---|
| `location` | `fbs::model_location_type_t` | `MODEL_LOCATION_IN_FLASH_RODATA` | 模型来源（rodata/分区/SD 卡） |
| `max_internal_size` | `int`（字节） | `0` | 限制内部 RAM 上限；超出落到 PSRAM。`0` 表示不限内部 RAM（有 PSRAM 时优先 PSRAM） |
| `mm_type` | `memory_manager_t` | `MEMORY_MANAGER_GREEDY` | 内存管理器（当前仅 GREEDY 实现） |
| `key` | `const uint8_t *` | `nullptr` | 加密模型密钥 |
| `param_copy` | `bool` | `true` | 是否把参数从 FLASH 拷到更快内存；`false` 省 RAM 但推理变慢（仅 rodata/partition 生效） |
| `input_shapes` | `std::map<std::string, std::vector<int>>` | `{}` | 覆盖图输入形状；空则用模型内形状 |

## 四、ESP-PPQ 量化参数（`espdl_quantize_onnx` / `espdl_quantize_torch`）

| 参数 | 说明 | 取值 |
|---|---|---|
| `target` | 量化目标平台（决定量化策略 + 舍入） | `"c"` / `"esp32s3"` / `"esp32p4"` |
| `num_of_bits` | 量化位宽 | `8` 或 `16` |
| `calib_steps` | 校准步数 | 整数，如 `32` |
| `input_shape` | 输入形状（不含 batch） | 如 `[1,1]`、`[3,224,224]`；batch 固定 1 |
| `inputs` | 具体测试输入；`None` 则用随机 | 张量或 `None` |
| `error_report` | 是否打印量化误差报告 | bool |
| `skip_export` | 是否跳过导出（只评估） | bool |
| `export_test_values` | 是否把测试输入/输出写入 `.espdl`（板端 `test()` 需要） | bool |
| `dispatching_override` | 量化调度覆盖 | 通常 `None` |
| `device` | 运行设备 | `"cpu"` / `"cuda"` |
| `verbose` | 日志级别 | 整数 |

仅 `espdl_quantize_torch` 额外支持流式相关参数：

| 参数 | 说明 |
|---|---|
| `auto_streaming` | 开启自动流式转换（插入 StreamingCache） |
| `streaming_input_shape` | 每个 chunk 的输入形状（时间维较小） |
| `streaming_table` | 手动 cache 配置列表（`insert_streaming_cache_on_var` 生成） |

## 五、AutoQuant 搜索设置（`AutoQuantSearchSetting`）

| 字段 | 默认 | 说明 |
|---|---|---|
| `search_mode` | `"exhaustive"` | `"exhaustive"` 全枚举；`"fast"` 先筛后搜 |
| `num_of_candidates` | `5` | 最终保留 Top-K |
| `score_direction` | `"maximize"` | 精度/mAP 用 maximize；误差/loss 用 minimize |
| `candidate_filter` | `low_latency_candidate_filter` | Top-K 过滤；纯按分排序设 `None` |
| `run_dir` | `"outputs/auto_quant"` | 结果目录 |
| `resume` | `False` | 从已有 `run_dir` 续搜 |
| `top_strategy`（fast） | `10` | 快筛后保留策略组数 |
| `early_stop_patience`（fast） | `3` | 连续 N 次无提升则停止 |
| `strategy_space` | — | 控制各量化策略是否参与搜索（如 `mixed_precision`） |
| `param_space` | — | 控制各策略参数候选（如 `calib_algorithm.method = ["kl","mse"]`） |

`evaluate_fn` 约束：返回 `(score, extras)`；`score` 为有限数；`extras` 为 dict 且不含保留键 `score/hash/index/folder/files/strategy/params`。

## 六、平台与舍入策略（来自 `operator_support_state.md`）

| 平台类别 | 芯片 | PIE | 舍入 | Conv/Gemm 量化 |
|---|---|---|---|---|
| 无 PIE（C 实现） | ESP32 / C2 / C3 / C5 / C6 / S2 | 无 | round-half-up | per-tensor |
| PIE V1 | ESP32-S3 | V1 | round-half-up | per-tensor |
| PIE V2 | ESP32-P4 / ESP32-S31 | V2 | round-half-even | Conv/Gemm per-channel，其余 per-tensor |

> ESP32-P4 per-channel 需 ESP-PPQ >= 1.2.10 且 ESP-DL >= 3.3.1；向后兼容旧 per-tensor 模型。
