# ESP-DL 常见陷阱汇总

> 汇总自仓库文档（`docs/en/tutorials/`）、示例与头文件。每条都是真实��为，不是推测。

## 1. ESP-DL 不直接跑 ONNX/PyTorch
只能跑 `.espdl`。先用 ESP-PPQ（`espdl_quantize_onnx` / `espdl_quantize_torch`）量化导出。TensorFlow/Paddle 需先转 ONNX。

## 2. `target` 必须与部署芯片一致
`c`（ESP32/C 系列，C 实现，round-half-up，per-tensor）、`esp32s3`（PIE V1，round-half-up，per-tensor）、`esp32p4`（PIE V2，round-half-even，Conv/Gemm per-channel）。混用会导致推理结果不准。`idf.py set-target` 也要对应。

## 3. 输入必须量化，输出必须反量化
8-bit 模型输入 `int8_t`，16-bit 输入 `int16_t`。float → int 用 `dl::quantize<T>(v, DL_RESCALE(exp))`；int → float 用 `dl::dequantize(q, DL_SCALE(exp))`。`Scale = 2^exp`。
- **易错**：`quantize` 第二参是 inverse scale（用 `DL_RESCALE`），`dequantize` 第二参是 scale（用 `DL_SCALE`）。

## 4. 内存规划器复用 buffer
模型的输入/中间/输出共享一块规划好的内存。`run()` 后再读 `model_input->data` 可能已被输出或中间张量覆盖。所需数据应在再次 `run()` 前读取或拷出。

## 5. 16 字节对齐强制要求
`TensorBase` 首地址必须 16 字节对齐，内存大小须为 16 字节倍数。`.espdl` 嵌入 rodata 必须用 `target_add_aligned_binary_data`（而非 `EMBED_FILES`）。`.info` 里的测试值也 16 字节对齐（不足补 0）。

## 6. rodata 符号名规则
`extern const uint8_t <name>[] asm("_binary_<filename_dots_to_underscores>_start");`。文件名中的 `.` 要换成 `_`，例如 `model.espdl` → `_binary_model_espdl_start`。

## 7. 分区模型 label 三处一致
`partition.csv` 的 Name（≤16 字符含 `\0`）、构造函数第 1 参、`esptool_py_flash_to_partition` 第 2 参必须相同。分区 SubType 必须是 `spiffs`，Type 为 `data`。

## 8. `test()` 需要测试值
需在 ESP-PPQ 导出时设 `export_test_values=True`。部署时可重新导出 `export_test_values=False` 以减小体积。INT16 模型 `test()` 允许每元素 ±1（量化舍入）。

## 9. 仅 batch_size = 1
不支持多 batch 或动态 batch。量化与部署都按 batch=1。

## 10. 默认单核，要双核需显式指定
`run()` 默认 `RUNTIME_MODE_SINGLE_CORE`。想用第二核（Conv2D/DepthwiseConv2D）传 `RUNTIME_MODE_MULTI_CORE` 或 `RUNTIME_MODE_AUTO`。

## 11. `param_copy=false` 的代价
保持参数在 FLASH 可省 PSRAM/内部 RAM，但推理变慢（FLASH 比 PSRAM 慢）。仅 `MODEL_LOCATION_IN_FLASH_RODATA`（且未设 `CONFIG_SPIRAM_RODATA`）或 `MODEL_LOCATION_IN_FLASH_PARTITION` 生效。

## 12. ESP32（无 PIE）很慢
`target="c"` 的算子是纯 C 实现，无 PIE 加速。AI 工作负载优先 ESP32-S3（PIE V1）或 ESP32-P4（PIE V2）。

## 13. 校准 dataloader 必须 `shuffle=False`
计算量化误差会多次遍历数据集，`shuffle=True` 会得到错误误差统计。

## 14. Resize 算子只支持 int8
`Resize` 不支持 int16 / float32；支持 1d/2d nearest/linear/bilinear，不支持 roi 与 antialias。

## 15. Conv 等算子用 NHWC/NWC 布局
为利用指令级加速，Conv、GlobalAveragePool、AveragePool、MaxPool、Resize 的输入/输出采用 NHWC 或 NWC 布局（与 ONNX/PyTorch 默认不同），ESP-PPQ 导出时已处理，自行预处理图像时注意。

## 16. 不同平台 `.espdl` 不可混用
平台间的量化策略与舍入不同，混用推理结果不准（见第 2 条）。

## 17. 大模型嵌入 rodata 需扩 app 分区
`.rodata` 模型随 app 一起烧，大模型可能超 app 分区；解决：增大 app 分区、改分区加载或 SD 卡加载。

## 18. SD 卡加载比 FLASH 慢
SD 卡模型数据需拷到 RAM，加载更慢；但省 Flash、方便换模型。SD 卡应 FAT32，否则挂载时自动格式化会丢数据。

## 19. 算子支持有限
当前 64 个算子已实现测试（见 `operator_support_state.md`），部分算子不实现全部属性。导出 ONNX 推荐 opset 18。

## 20. 激活函数用 8-bit LUT
除 ReLU 和 PReLU 外，所有激活函数用 8-bit LUT 实现，任意激活复杂度基本相同。无需为“速度”手动替换 sigmoid/tanh。
