# USB 主机：UAC 主机驱动（USB 音频录放）

> **适用摘要**: 用 UAC 主机驱动连接 USB 音频设备（扬声器、麦克风、耳机），完成 12 步生命周期：安装 → 连接回调 → 打开 → 查格式（alt 参数）→ start/stop 流 → suspend/resume → 音量/静音控制 → RX/TX_DONE 回调 → close → uninstall。当前支持 UAC 1.0。

## 触发意图

- "USB 音频"
- "UAC 主机"
- "usb host uac"
- "USB 扬声器 / USB 喇叭"
- "USB 麦克风"
- "uac_host"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件依赖 | `idf.py add-dependency "espressif/usb_host_uac^1.5.0"`（会带 `usb`） |
| ESP-IDF | `>= 5.5.3` |
| 兼容性 | 仅 UAC 1.0 设备 |
| 参考头文件 | `host/class/uac/usb_host_uac/include/usb/uac_host.h`、`uac.h` |

> 一个同时含麦克风和扬声器的物理设备，驱动会拆成两个**逻辑设备**：分别触发 `UAC_HOST_DRIVER_EVENT_RX_CONNECTED`（麦克风）与 `UAC_HOST_DRIVER_EVENT_TX_CONNECTED`（扬声器）。

## 分步说明

### 1. 安装 USB Host Library 与 UAC 驱动

```c
#include "usb/usb_host.h"
#include "usb/uac_host.h"

static void uac_host_lib_cb(uint8_t addr, uint8_t iface_num,
                            const uac_host_driver_event_t event, void *arg) {
    // 设备连入：addr/iface_num 给 open() 用，event 区分 RX(TX)
    // 典型做法：把 (addr, iface_num, event) 投递到应用队列异步处理
}

// app_main 或任务中：
const usb_host_config_t host_config = {
    .skip_phy_setup = false,
    .intr_flags = ESP_INTR_FLAG_LOWMED,
};
ESP_ERROR_CHECK(usb_host_install(&host_config));

const uac_host_driver_config_t uac_config = {
    .create_background_task = true,
    .task_priority = 5,
    .stack_size    = 4096,
    .core_id       = 0,                 // 或 tskNO_AFFINITY
    .callback      = uac_host_lib_cb,
    .callback_arg  = NULL,
};
ESP_ERROR_CHECK(uac_host_install(&uac_config));
```

> 仍需自建 Daemon Task 调 `usb_host_lib_handle_events()`（见 `host_library_basic.md`）。`create_background_task=false` 时应用须周期调 `uac_host_handle_events(timeout)`。

### 2. 打开 UAC 设备

`addr` 与 `iface_num` 来自驱动事件回调。`buffer_size` 是驱动内部环形缓冲；`buffer_threshold` 是触发 `RX_DONE` / `TX_DONE` 的水位。

```c
const uac_host_device_config_t dev_config = {
    .addr              = addr,      // 来自 RX/TX_CONNECTED 事件
    .iface_num         = iface_num,
    .buffer_size       = 16000,     // 内部环形缓冲字节数
    .buffer_threshold  = 4000,      // 水位：达到则触发回调
    .callback          = uac_device_cb,
    .callback_arg      = NULL,
};
uac_host_device_handle_t uac_dev = NULL;
ESP_ERROR_CHECK(uac_host_device_open(&dev_config, &uac_dev));
```

### 3. 查询设备信息与可用音频格式

UAC 设备用一组“备用设置”（alt interface）列举支持的格式。alt 从 1 开始。`uac_host_get_device_alt_param()` 给出每个 alt 的采样格式、声道、位深、采样率（离散数组或连续区间）。

```c
uac_host_dev_info_t info;
ESP_ERROR_CHECK(uac_host_get_device_info(uac_dev, &info));
// info.type == UAC_STREAM_TX(扬声器) / UAC_STREAM_RX(麦克风)
// info.iface_alt_num: 备用设置总数

uac_host_dev_alt_param_t p;
ESP_ERROR_CHECK(uac_host_get_device_alt_param(uac_dev, 1, &p));
// p.format == 1 (PCM)
// p.channels, p.bit_resolution, p.subframe_size
// p.sample_freq_type == 0: 连续区间 sample_freq_lower ~ sample_freq_upper
// p.sample_freq_type  > 0: 离散 sample_freq[0..type-1]

// 打印全部 alt 调试
uac_host_printf_device_param(uac_dev);
```

### 4. 启动音频流

`uac_host_stream_config_t` 给出本次流的声道/位深/采样率。驱动会自动选匹配的 alt 设置。

```c
const uac_host_stream_config_t stream_config = {
    .channels       = p.channels,         // 取自 alt 参数
    .bit_resolution = p.bit_resolution,
    .sample_freq    = 48000,              // 必须是该 alt 支持的频率
};
ESP_ERROR_CHECK(uac_host_device_start(uac_dev, &stream_config));

// 可选：启动后先挂在“已挂起”状态，稍后 resume 才真正传输
// 在 stream_config.flags 里设 FLAG_STREAM_SUSPEND_AFTER_START
```

### 5. 音频数据收发

- **扬声器（TX）**：应用写 PCM，驱动异步入环形缓冲再 USB 发送。
- **麦克风（RX）**：USB 收到的数据入环形缓冲，水位达 `buffer_threshold` 时触发 `UAC_HOST_DEVICE_EVENT_RX_DONE`，应用再 `uac_host_device_read()` 取走。

```c
// TX：往扬声器写 PCM（本例：循环写 2048 字节块）
uint8_t chunk[2048];
esp_err_t ret = uac_host_device_write(uac_dev, chunk, sizeof(chunk),
                                      pdMS_TO_TICKS(1000));

// RX：从麦克风读
uint32_t bytes_read = 0;
uac_host_device_read(uac_dev, mic_buf, mic_buf_size, &bytes_read, 0);
```

### 6. 设备事件回调

不要在回调里做重活——把它投递到应用队列异步处理（示例 `audio_player` 即此模式）。

```c
static void uac_device_cb(uac_host_device_handle_t uac_device_handle,
                          const uac_host_device_event_t event, void *arg) {
    switch (event) {
    case UAC_HOST_DEVICE_EVENT_RX_DONE:        // 麦克风缓冲到水位，可 read
        break;
    case UAC_HOST_DEVICE_EVENT_TX_DONE:        // 扬声器缓冲低于水位，可 write
        break;
    case UAC_HOST_DEVICE_EVENT_TRANSFER_ERROR:
        break;
    case UAC_HOST_DRIVER_EVENT_DISCONNECTED:   // 设备拔出：必须 close
        uac_host_device_close(uac_device_handle);
        break;
    default: break;
    }
}
```

### 7. 挂起/恢复、音量、静音

```c
uac_host_device_suspend(uac_dev);   // 暂停传输（不释放资源）
uac_host_device_resume(uac_dev);    // 恢复，沿用原 stream_config

uac_host_device_set_mute(uac_dev, false);            // false=取消静音
bool muted; uac_host_device_get_mute(uac_dev, &muted);

uac_host_device_set_volume(uac_dev, 50);             // 0..100 归一化音量
uint8_t vol; uac_host_device_get_volume(uac_dev, &vol);

uac_host_device_set_volume_db(uac_dev, 0x0100);      // 单位 1/256 dB（0x0100 = 1 dB）
int16_t db; uac_host_device_get_volume_db(uac_dev, &db);
```

> 设备若不支持音量/静音控制，返回 `ESP_ERR_NOT_SUPPORTED`。

### 8. 停止、关闭、卸载

```c
uac_host_device_stop(uac_dev);     // 停流并释放传输资源
uac_host_device_close(uac_dev);    // 关闭设备
// 所有设备 close 后才能卸载：
uac_host_uninstall();
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 收不到 TX/RX_CONNECTED | UAC 2.0 设备 | 驱动仅支持 UAC 1.0；换 1.0 设备 |
| `ESP_ERR_NOT_FOUND` start 失败 | stream_config 不匹配任何 alt | 先 `get_device_alt_param` 查支持的声道/位深/采样率 |
| `ESP_ERR_NOT_SUPPORTED` set_volume/mute | 设备无该控制功能 | 查设备能力，代码里容忍 `ESP_ERR_NOT_SUPPORTED` |
| 音频断续 / dropouts | ISOC URB 太少或缓冲太小 | 增大 `buffer_size`；调高 `CONFIG_UAC_NUM_ISOC_URBS`（默认 3）/ `CONFIG_UAC_NUM_PACKETS_PER_URB`（默认 3） |
| `RX_DONE` 不触发 | `buffer_threshold` 过大 | `buffer_threshold` 设为 `buffer_size` 的约 1/4 |
| `ESP_ERR_INVALID_STATE` uninstall | 有设备未关闭 | 对每个 handle 调 `close` 后再 `uninstall` |
| 扬声器/麦克风互相干扰 | 同时收发抢占 USB 带宽 | 录音时先 `uac_host_device_suspend` 麦克风再播 PCM，播完再 `resume`（见 audio_player 示例） |
| 回调里直接做重活阻塞 | 阻塞内部事件分发 | 回调只入队，在另一任务 read/write |

## 参考

- `host/class/uac/usb_host_uac/include/usb/uac_host.h` — UAC 主机 API、`uac_host_device_config_t`、`uac_host_stream_config_t`、`FLAG_STREAM_SUSPEND_AFTER_START`
- `host/class/uac/usb_host_uac/Kconfig` — `CONFIG_UAC_NUM_ISOC_URBS`、`CONFIG_UAC_NUM_PACKETS_PER_URB`、`CONFIG_UAC_FREQ_NUM_MAX`、`CONFIG_UAC_DEV_ADDR_LIST_MAX`
- `host/class/uac/usb_host_uac/README.md` — 完整 12 步 API 生命周期
- 示例：`host/class/uac/usb_host_uac/examples/audio_player/`（`main/main.c`：扬声器 siren 播放 + 麦克风录音回放、alt 参数查找匹配采样率、suspend/resume 切换）
- 相关（esp-iot-solution）：`examples/usb/host/usb_audio_player`（README 引用）
