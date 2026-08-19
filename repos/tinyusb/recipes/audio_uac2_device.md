# Audio 设备类（UAC2 麦克风/扬声器/耳机）

> **适用摘要**: 用 TinyUSB Audio 类实现 USB 音频 2.0（UAC2）设备，包括麦克风（IN 端点）、扬声器（OUT 端点 + 反馈端点）、耳机（双向）、异步反馈端点（feedback endpoint）、采样率协商与音量/静音控制。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/tinyusb/resources/`, source/examples in `repos/tinyusb/`, and this recipe path `repos/tinyusb/recipes/audio_uac2_device.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "USB 音频设备"
- "TinyUSB UAC2 / UAC2 headset / speaker / mic"
- "tud_audio_write / tud_audio_read"
- "USB 麦克风 / USB DAC / USB 喇叭"
- "audio feedback endpoint / feedback endpoint"
- "采样率协商 / sample rate negotiation"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/device/uac2_headset/`、`examples/device/uac2_speaker_fb/`、`examples/device/audio_4_channel_mic/`、`examples/device/audio_test_multi_rate/`、`examples/device/cdc_uac2/` |
| 配置项 | `CFG_TUD_AUDIO=1`、`CFG_TUD_AUDIO_ENABLE_EP_IN/OUT`、`CFG_TUD_AUDIO_ENABLE_FEEDBACK_EP`、`CFG_TUD_AUDIO_FUNC_1_EP_IN/OUT_SW_BUF_SZ` |
| 头文件 | `src/class/audio/audio_device.h`（API）、`src/class/audio/audio.h`（UAC1/UAC2 描述符宏与控制结构）、`src/device/usbd.h`（`TUD_AUDIO_EP_SIZE`） |

## 分步说明

### 1. tusb_config.h 使能 Audio（IN/OUT + feedback）

Audio 配置项最多（端点尺寸要按采样率/位深/通道数算），下面是扬声器 + 反馈端点的最小集（取自 `uac2_speaker_fb/src/tusb_config.h`）：

```c
#define CFG_TUD_AUDIO             1

// 打开 OUT（主机->设备，即扬声器方向）端点
#define CFG_TUD_AUDIO_ENABLE_EP_OUT                1
// 异步 sink 必须开 feedback 端点（告诉主机真实消费速率）
#define CFG_TUD_AUDIO_ENABLE_FEEDBACK_EP           1

// 通道 / 位深 / 采样率（OUT 方向）
#define CFG_TUD_AUDIO_FUNC_1_N_CHANNELS_RX              2
#define CFG_TUD_AUDIO_FUNC_1_N_BYTES_PER_SAMPLE_RX      2   // 16bit 装在 16bit slot
#define CFG_TUD_AUDIO_FUNC_1_RESOLUTION_RX              16

// 端点最大尺寸：FS 与 HS 分别按公式算，再取最大
#define CFG_TUD_AUDIO_FUNC_1_MAX_SAMPLE_RATE_FS     48000
#define CFG_TUD_AUDIO_FUNC_1_EP_OUT_SZ_FS           TUD_AUDIO_EP_SIZE(false, 48000, 2, 2)
#define CFG_TUD_AUDIO_FUNC_1_MAX_SAMPLE_RATE_HS     96000
#define CFG_TUD_AUDIO_FUNC_1_EP_OUT_SZ_HS           TUD_AUDIO_EP_SIZE(true,  96000, 2, 2)
#define CFG_TUD_AUDIO_FUNC_1_EP_OUT_SZ_MAX          TU_MAX(CFG_TUD_AUDIO_FUNC_1_EP_OUT_SZ_FS, CFG_TUD_AUDIO_FUNC_1_EP_OUT_SZ_HS)

// 软件 FIFO：FIFO_COUNT 反馈法要求 >= 4 倍 EP size（HS 下每 1ms 读一次则要 8 倍）
#define CFG_TUD_AUDIO_FUNC_1_EP_OUT_SW_BUF_SZ       TU_MAX(4 * CFG_TUD_AUDIO_FUNC_1_EP_OUT_SZ_FS, 32 * CFG_TUD_AUDIO_FUNC_1_EP_OUT_SZ_HS)
```

> **方向记法**：`TX` = 设备→主机（麦克风，IN 端点）；`RX` = 主机→设备（扬声器，OUT 端点）。配置宏里的 `_TX_` 对应 IN 端点，`_RX_` 对应 OUT 端点，与端点方向而非"收发语义"对应。

### 2. 端点尺寸公式（`TUD_AUDIO_EP_SIZE`）

`usbd.h` 提供唯一尺寸计算宏，参数为 `(是否高速, 采样率, 每样本字节数, 通道数)`：

```c
// src/device/usbd.h
#define TUD_AUDIO_EP_SIZE(_is_highspeed, _maxFrequency, _nBytesPerSample, _nChannels) \
    (((((_maxFrequency) + ((_is_highspeed) ? 7999 : 999)) / ((_is_highspeed) ? 8000 : 1000)) + 1) \
     * (_nBytesPerSample) * (_nChannels))
```

FS：每 1ms 一帧；HS：每 125us 一微帧（8 帧/ms）。`+1` 是为应付非整数取整的那一帧（如 44.1kHz）。

### 3. 麦克风（IN 端点）：周期性 `tud_audio_write`

麦克风方向用 IN 端点，应用按音频时钟（I2S 接收回调或定时器）周期性把样本推进软件 FIFO（取自 `audio_4_channel_mic/src/main.c`）：

```c
// 每帧字节数 = 采样率/1000 * 每样本字节数 * 通道数（FS，1ms 一帧）
void audio_task(void) {
  static uint32_t start_ms = 0;
  uint32_t curr_ms = board_millis();
  if (start_ms == curr_ms) return;     // 没到下一帧
  start_ms = curr_ms;

  tud_audio_write(i2s_dummy_buffer,
                  CFG_TUD_AUDIO_FUNC_1_SAMPLE_RATE / 1000
                  * CFG_TUD_AUDIO_FUNC_1_N_BYTES_PER_SAMPLE_TX
                  * CFG_TUD_AUDIO_FUNC_1_N_CHANNELS_TX);
}
```

> 真实应用里应在 I2S DMA 接收完成回调里调 `tud_audio_write()`，而不是在主循环里按 `board_millis()` 节拍——后者只是示例的简化。

### 4. 扬声器（OUT 端点）：周期性 `tud_audio_read`

主机→设备的样本由栈放进软件 FIFO，应用按音频时钟取走（取自 `uac2_speaker_fb/src/main.c`）：

```c
void audio_task(void) {
  static uint32_t start_ms = 0;
  uint32_t curr_ms = board_millis();
  if (start_ms == curr_ms) return;
  start_ms = curr_ms;

  uint16_t length = (uint16_t)(current_sample_rate / 1000
                               * CFG_TUD_AUDIO_FUNC_1_N_BYTES_PER_SAMPLE_RX
                               * CFG_TUD_AUDIO_FUNC_1_N_CHANNELS_RX);

  // 44.1/88.2 kHz 非整数：每 10/5 ms 补一帧样本（示例里演示；真实 I2S 时钟驱动时不需要）
  if (current_sample_rate == 44100 && (curr_ms % 10 == 0)) {
    length += CFG_TUD_AUDIO_FUNC_1_N_BYTES_PER_SAMPLE_RX * CFG_TUD_AUDIO_FUNC_1_N_CHANNELS_RX;
  }
  tud_audio_read(i2s_dummy_buffer, length);
}
```

### 5. 异步反馈端点（feedback endpoint）

异步 sink（扬声器）必须用反馈端点告诉主机"我真实消费得多快"，否则主机会上溢/下溢。TinyUSB 提供三种反馈法（见 `audio_device.h` 枚举）：

```c
enum {
  AUDIO_FEEDBACK_METHOD_DISABLED,
  AUDIO_FEEDBACK_METHOD_FREQUENCY_FIXED,   // SOF ISR 里按主时钟周期数算（需 mclk_freq）
  AUDIO_FEEDBACK_METHOD_FREQUENCY_FLOAT,
  AUDIO_FEEDBACK_METHOD_FREQUENCY_POWER_OF_2,
  AUDIO_FEEDBACK_METHOD_FIFO_COUNT         // 驱动内部按 FIFO 水位调节（最稳，需 FIFO >= 4*EP size）
};
```

最常用的是 **FIFO_COUNT**（无需 SOF ISR，跨平台稳定，`uac2_speaker_fb` 即用此法）。只需实现参数回调：

```c
// 主机选中 streaming 接口时栈来问反馈参数
void tud_audio_feedback_params_cb(uint8_t func_id, uint8_t alt_itf,
                                  audio_feedback_params_t *feedback_param) {
  (void)func_id; (void)alt_itf;
  feedback_param->method     = AUDIO_FEEDBACK_METHOD_FIFO_COUNT;
  feedback_param->sample_freq = current_sample_rate;
  // fifo_threshold 不设则默认半 FIFO；可调小以降延迟
}
```

如选 `FREQUENCY_FIXED/FLOAT`，则需另在 SOF ISR 回调里调用 `tud_audio_feedback_update()` 提供主时钟周期数：

```c
// TU_ATTR_FAST_FUNC void tud_audio_feedback_interval_isr(uint8_t func_id,
//        uint32_t frame_number, uint8_t interval_shift);
// 该回调里读硬件主时钟计数器，再调：
//   tud_audio_feedback_update(func_id, cycles_since_last_sof);
// 返回值为 16.16 格式的 feedback（FS 设备驱动会自动转 10.14）
```

反馈值格式：HS 用 16.16，FS 名义上 10.14；但 **Windows UAC2 驱动有 bug，FS 也要求 16.16**，TinyUSB 已在驱动内部处理（见 `audio_device.h` 注释）。

### 6. 采样率协商（UAC2 Clock Unit）

UAC2 把采样率放在 Clock Source 实体上（不像 UAC1 放端点）。主机通过类特定 GET/SET RANGE/CUR 查询。`uac2_speaker_fb` 用 `tud_audio_get_req_entity_cb` 分发到 Clock / Feature Unit：

```c
bool tud_audio_get_req_entity_cb(uint8_t rhport, tusb_control_request_t const *p_request) {
  audio20_control_request_t const *request = (audio20_control_request_t const *)p_request;
  if (request->bEntityID == UAC2_ENTITY_CLOCK)        return audio20_clock_get_request(rhport, request);
  if (request->bEntityID == UAC2_ENTITY_FEATURE_UNIT) return audio20_feature_unit_get_request(rhport, request);
  return false;
}

// Clock Unit 返回支持的采样率范围（多采样率设备必备）
static bool audio20_clock_get_request(uint8_t rhport, audio20_control_request_t const *request) {
  if (request->bControlSelector == AUDIO20_CS_CTRL_SAM_FREQ
      && request->bRequest == AUDIO20_CS_REQ_RANGE) {
    const uint32_t sample_rates[] = {44100, 48000, 88200, 96000};
    audio20_control_range_4_n_t(TU_ARRAY_SIZE(sample_rates)) rangef = {
        .wNumSubRanges = tu_htole16(TU_ARRAY_SIZE(sample_rates))};
    for (uint8_t i = 0; i < TU_ARRAY_SIZE(sample_rates); i++) {
      rangef.subrange[i].bMin = (int32_t)sample_rates[i];
      rangef.subrange[i].bMax = (int32_t)sample_rates[i];
      rangef.subrange[i].bRes = 0;            // 离散值
    }
    return tud_audio_buffer_and_schedule_control_xfer(rhport, (tusb_control_request_t const *)request,
                                                       &rangef, sizeof(rangef));
  }
  // ... SAM_FREQ CUR / CLK_VALID CUR
}
```

> `tud_audio_buffer_and_schedule_control_xfer()` 把应答拷进控制缓冲（`CFG_TUD_AUDIO_CTRL_BUF_SZ`）并触发 EP0 发送；已有持久缓冲时可改用 `tud_control_xfer()` 省一次拷贝。

### 7. 音量/静音控制（Feature Unit）

Feature Unit 的 mute/volume 也是类特定控制请求。`tud_audio_version()` 返回 1（UAC1）或 2（UAC2），可在同一回调里分支处理（`uac2_speaker_fb` 做法）：

```c
bool tud_audio_set_req_entity_cb(uint8_t rhport, tusb_control_request_t const *p_request, uint8_t *buf) {
  (void)rhport;
  if (tud_audio_version() == 1) return audio10_set_req_entity(p_request, buf);
  if (tud_audio_version() == 2) return audio20_set_req_entity(p_request, buf);
  return false;
}
```

Volume 单位为 1/256 dB（UAC2 用 16-bit 有符号 `audio20_control_cur_2_t`；UAC1 同）。

### 8. 接口切换 / 流开始与停止

主机选 alternate setting 来开关流。两个回调各处理"开始"和"关闭"：

```c
// 主机选中非零 alt（开流）
bool tud_audio_set_itf_cb(uint8_t rhport, tusb_control_request_t const *p_request) {
  uint8_t itf = tu_u16_low(tu_le16toh(p_request->wIndex));
  uint8_t alt = tu_u16_low(tu_le16toh(p_request->wValue));
  if (ITF_NUM_AUDIO_STREAMING == itf && alt != 0) {
    blink_interval_ms = BLINK_STREAMING;   // 例：点亮 LED
  }
  return true;
}

// 主机切回 alt 0（关流 / 关端点）
bool tud_audio_set_itf_close_ep_cb(uint8_t rhport, tusb_control_request_t const *p_request) {
  uint8_t itf = tu_u16_low(tu_le16toh(p_request->wIndex));
  uint8_t alt = tu_u16_low(tu_le16toh(p_request->wValue));
  if (ITF_NUM_AUDIO_STREAMING == itf && alt == 0) {
    blink_interval_ms = BLINK_MOUNTED;
  }
  return true;
}
```

### 9. UAC2 音频回调速查

| 回调 | 何时触发 | 必需性 |
|---|---|---|
| `tud_audio_set_itf_cb` | 主机 SET_INTERFACE 选非零 alt（开流） | 推荐 |
| `tud_audio_set_itf_close_ep_cb` | 主机 SET_INTERFACE 关端点 | 推荐 |
| `tud_audio_set_req_entity_cb` | 类特定 SET 给 entity（Clock/Feature Unit） | 支持 UAC2 控制则必需 |
| `tud_audio_get_req_entity_cb` | 类特定 GET 给 entity | 同上 |
| `tud_audio_set_req_ep_cb` / `tud_audio_get_req_ep_cb` | 类特定请求给端点（UAC1 采样率在此） | UAC1 必需 |
| `tud_audio_feedback_params_cb` | 选反馈参数（method/sample_freq） | 开 feedback EP 时必需 |
| `tud_audio_feedback_interval_isr` | SOF 周期，用于 FREQUENCY_* 法算 feedback | 仅 FREQUENCY 法 |
| `tud_audio_rx_done_isr` | OUT 端点收到一包（ISR） | 调试用，正常不用 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 设备识别但无声 / 主机不收音 | IN/OUT 端点方向配反 | TX→IN 端点（麦克风），RX→OUT 端点（扬声器）；配置宏后缀 `_TX_`/`_RX_` 与端点方向一致 |
| 杂音、断续、下溢/上溢 | 反馈端点未开或反馈值不对 | 扬声器必须开 `CFG_TUD_AUDIO_ENABLE_FEEDBACK_EP=1` 并实现 `tud_audio_feedback_params_cb`；优先用 `AUDIO_FEEDBACK_METHOD_FIFO_COUNT` |
| FIFO_COUNT 法仍抖 | 软件 FIFO 太小 | `*_EP_*_SW_BUF_SZ >= 4 * EP size`（HS 下若每 1ms 读一次要 `32 * EP size`） |
| 采样率切换无效 | UAC2 没实现 Clock Unit 的 RANGE/CUR | 实现 `tud_audio_get_req_entity_cb` 处理 `UAC2_ENTITY_CLOCK` 的 `SAM_FREQ` RANGE/CUR |
| `#error EP software buffer size MUST BE at least as big as maximum EP size` | SW_BUF_SZ 小于 EP_SZ_MAX | SW 缓冲必须 ≥ 最大 EP 尺寸（多 alt 取所有 alt 的最大） |
| Windows 下 FS 设备 feedback 不工作 | Windows UAC2 驱动 bug，要求 16.16 | 驱动已自动按 16.16 发；确保别手动改格式 |
| 多采样率设备某个速率爆音 | 端点尺寸按低速率算 | 用 `TUD_AUDIO_EP_SIZE` 对每个 alt 算，`*_EP_*_SZ_MAX` 取所有 alt 的最大 |
| 描述符里缺反馈端点 | 只建了数据端点 | UAC2 描述符模板 `TUD_AUDIO20_HEADSET_STEREO_DESC` / 对应反馈宏需含 FB 端点（见各示例 `usb_descriptors.c`） |

## 参考

- `examples/device/uac2_headset/` — UAC2 耳机（扬声器+麦克风，双向，含音量控制）
- `examples/device/uac2_speaker_fb/` — UAC2 喇叭 + 反馈端点（FIFO_COUNT 法，含 UAC1/UAC2 双协议与 HID 调试）
- `examples/device/audio_4_channel_mic/` — 4 通道麦克风（IN 端点多通道）
- `examples/device/audio_test_multi_rate/` — 多采样率（Clock Unit RANGE 协商）
- `examples/device/audio_test/`、`examples/device/audio_test_freertos/` — 基础 UAC2 测试 + FreeRTOS 版
- `examples/device/cdc_uac2/` — CDC + UAC2 组合设备
- `src/class/audio/audio_device.h` — Audio 设备 API、回调与 `audio_feedback_params_t`、反馈法枚举
- `src/class/audio/audio.h` — UAC1/UAC2 描述符宏、控制结构（`audio20_control_range_4_n_t` 等）
- `src/device/usbd.h` — `TUD_AUDIO_EP_SIZE()` 端点尺寸宏
