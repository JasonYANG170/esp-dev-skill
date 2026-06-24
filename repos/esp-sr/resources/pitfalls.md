# ESP-SR 常见踩坑汇总

> 来源：仓库 `docs/`、头文件注释、`test_apps/` 与 README。每条都给出错误写法与正确写法。代码/API 名均为真实符号。

## 1. 用了 V1.* 已移除的接口

V2.0 删除了 `AFE_CONFIG_DEFAULT()`、`ESP_AFE_SR_HANDLE`、`ESP_AFE_VC_HANDLE`、`create()`（改名 `create_from_config`）。

```c
// ❌
afe_config_t *cfg = AFE_CONFIG_DEFAULT("MMNR", models, AFE_INTERNAL);
const esp_afe_sr_iface_t *h = ESP_AFE_SR_HANDLE;

// ✅
afe_config_t *cfg = afe_config_init("MMNR", models, AFE_TYPE_SR, AFE_MODE_HIGH_PERF);
const esp_afe_sr_iface_t *h = esp_afe_handle_from_config(cfg);
esp_afe_sr_data_t *d = h->create_from_config(cfg);
```

## 2. 没配 model 分区 / 没烧模型

`partitions.csv` 没有 `model` 行 → 编译报 `Failed to find model in partition table`；烧了应用但没烧模型 → `esp_srmodel_init` 返回空。

```makefile
# ✅ partitions.csv
model,    data,         ,        , 6000K
```

烧模型必须 `idf.py flash`（非 `app-flash`）。

## 3. feed 帧长/通道写死

```c
// ❌
int16_t buff[512];
afe_handle->feed(afe_data, buff);

// ✅
int cs  = afe_handle->get_feed_chunksize(afe_data);
int nch = afe_handle->get_feed_channel_num(afe_data);
int16_t *buff = malloc(cs * sizeof(int16_t) * nch);
```

## 4. MultiNet 吃了错误的数据源

MultiNet 只接受 AFE fetch 出来的**单声道 16k/16bit**数据，且帧长 = `get_fetch_chunksize`。

```c
// ❌ 喂多通道原始 I2S
multinet->detect(md, multich_i2s);

// ✅
afe_fetch_result_t *res = afe_handle->fetch(afe_data);
multinet->detect(md, res->data);
```

## 5. 命令词改了没 update

```c
// ❌
esp_mn_commands_add(1, "da kai kong tiao");

// ✅
esp_mn_commands_add(1, "da kai kong tiao");
esp_mn_error_t *err = esp_mn_commands_update();
if (err) { /* err->phrases 是无法解析的项 */ }
```

## 6. 唤醒态判断字段没用对

```c
// ❌ 忽略 fetch 返回
afe_handle->fetch(afe_data);

// ✅
afe_fetch_result_t *res = afe_handle->fetch(afe_data);
if (res->wakeup_state == WAKENET_DETECTED) {
    // res->wake_word_index, res->trigger_channel_id
}
```

## 7. 阈值范围搞混

- WakeNet（独立 / AFE）：**0.4 ~ 0.9999**
- MultiNet：**0.0 ~ 0.9999**
- VADNet：**0.5 ~ 0.9999**

```c
// ❌ WakeNet 阈值 0.1（低于下限）
wakenet->set_det_threshold(md, 0.1, 1);

// ✅
wakenet->set_det_threshold(md, 0.8, 1);
```

## 8. AEC 缓冲区未对齐

`aec_process` / `afe_aec_process` 的 mic/ref/out 必须 16 字节对齐。

```c
// ❌
int16_t *out = malloc(frame_size * 2);

// ✅
int16_t *out = heap_caps_aligned_alloc(16, frame_size * sizeof(int16_t), MALLOC_CAP_8BIT);
```

## 9. VAD“吃字”没处理 cache

```c
// ❌
fwrite(res->data, 1, res->data_size, fp);

// ✅
if (res->vad_cache_size > 0)
    fwrite(res->vad_cache, 1, res->vad_cache_size, fp);
fwrite(res->data, 1, res->data_size, fp);
```

## 10. menuconfig 没选模型导致 NULL 句柄

```c
// ❌ 直接用，NULL 崩溃
char *name = esp_srmodel_filter(models, ESP_WN_PREFIX, NULL);
wakenet->create(name, DET_MODE_95);

// ✅ 判空
char *name = esp_srmodel_filter(models, ESP_WN_PREFIX, NULL);
if (!name) { ESP_LOGE(TAG, "select wakenet in menuconfig & flash"); return; }
```

## 11. 内存泄漏（不 destroy/deinit）

```c
// ❌ 反复 create
while (1) { afe_handle->create_from_config(cfg); }

// ✅
esp_afe_sr_data_t *d = afe_handle->create_from_config(cfg);
/* ... */
afe_handle->destroy(d);
afe_config_free(cfg);
esp_srmodel_deinit(models);
```

## 12. C 系列选了 wn9

C3/C5/C6 无 PSRAM/SIMD，只能跑 `wn9s_*`（Depthwise Separable Conv）。

```
# ❌ ESP32-C5
Select WakeNet -> wn9_hilexin

# ✅
Select WakeNet -> wn9s_hilexin
```

## 13. 命令词含数字/特殊字符/中英混

```c
// ❌ 解析失败
esp_mn_commands_add(1, "TURN ON THE LIGHT 123");
esp_mn_commands_add(2, "关闭电灯？");

// ✅
esp_mn_commands_add(1, "TURN ON THE LIGHT");
esp_mn_commands_add(2, "guan bi dian deng");
```

同一模型内**不可中英混**。

## 14. 8kHz 输入用了 AFE_TYPE_VC

`AFE_TYPE_VC` 期望 16kHz；8kHz 必须用 `AFE_TYPE_VC_8K`，否则内部重采样出错。独立 `aec_create` 仅支持 16000。

## 15. AFE 的 aec_mode 用了 FD/VOIP

AFE 内部 `aec_mode` 只接受 `AEC_MODE_SR_LOW_COST` / `AEC_MODE_SR_HIGH_PERF`。要 FD/VOIP 走 `AFE_TYPE_FD` / `AFE_TYPE_VC` 或独立 `aec_create`。

## 16. 引用了错误芯片目录的头

头文件按 `include/<target>/` 分目录，CMake 通过 `TARGET_LIB_PATH` 自动选。若手动指定 `-I`，确保与 `IDF_TARGET` 一致（如 `esp32s3`、`esp32p4`、`esp32p4_less_v3`）。

## 17. fetch 超时阻塞

`fetch()` 默认超时 2000ms；需要更短超时用 `fetch_with_delay(afe_data, ticks)`。

## 18. MultiNet 超时时间没设

`multinet->create(model_name, duration_ms)` 的 duration 决定 `ESP_MN_STATE_TIMEOUT` 触发时机；设太短会频繁超时退出，设太长用户说完很久才结束。测试用 6000ms。

## 19. ringbuf 繁忙未察觉

`res->ringbuff_free_pct > 0.5` 表示 feed 太快 / fetch 太慢，ringbuf 堆积。需提高 fetch 任务优先级或减慢 feed。

## 20. 引用 WebRTC VAD 的 `vad_process` 但选了 VADNet

WebRTC VAD 用 `esp_vad.h` 的 `vad_create/process`；VADNet 用 `esp_vadn_iface.h` 且通常经 AFE。两者 API 不通用，别混调。
