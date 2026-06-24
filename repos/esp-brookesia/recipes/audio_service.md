# Audio 服务：播放、控制、编解码与 AFE

> **适用摘要**: 使用 `brookesia_service_audio` 播放本地/网络音频、多 URL 队列、播放控制（暂停/恢复/停止）、编解码（PCM/OPUS/G711A）回环，以及 AFE（VAD + 唤醒词）。Audio 服务依赖 HAL 设备，需先初始化存储与音频设备。

## 触发意图

- "用 Brookesia 播放音频"
- "播放控制 暂停/恢复/停止"
- "录音编码 PCM/OPUS"
- "AFE 语音唤醒 VAD"
- "播放多个 URL"

## 前置条件

| 条件 | 要求 |
|---|---|
| 依赖 | `espressif/brookesia_service_audio` + HAL（`brookesia_hal_interface`/`brookesia_hal_adaptor`/`brookesia_hal_boards`）|
| 目标 | 板级工程：`idf.py gen-bmgr-config -b <board>`（需 `AudioCodecPlayer`/`AudioCodecRecorder`）|
| 参考示例 | `examples/service/audio` |

## 分步说明

### 1. 初始化 HAL 设备 + 启动 ServiceManager

```cpp
#include "brookesia/hal_interface.hpp"
#include "brookesia/hal_adaptor.hpp"
using AudioHelper = service::helper::Audio;

hal::init_device(hal::StorageDevice::DEVICE_NAME);
hal::init_device(hal::AudioDevice::DEVICE_NAME);

auto &service_manager = service::ServiceManager::get_instance();
service_manager.start();
BROOKESIA_CHECK_FALSE_EXIT(AudioHelper::is_available(), "Audio service is not available");
```

### 2. 启动前设置配置（配置须在 bind 之前）

```cpp
// 播放器任务配置
AudioHelper::PlaybackConfig playback{
    .player_task = {
        .core_id = 0,
        .priority = 5,
        .stack_size = 4 * 1024,
#if CONFIG_SPIRAM_XIP_FROM_PSRAM
        .stack_in_ext = true,    // 用 PSRAM 但不开 XIP 时，置 true 防止 Flash 操作导致崩溃
#else
        .stack_in_ext = false,
#endif
    }
};
AudioHelper::call_function_sync(
    AudioHelper::FunctionId::SetPlaybackConfig,
    BROOKESIA_DESCRIBE_TO_JSON(playback).as_object());

// 编/解码静态配置（默认值即可）
AudioHelper::EncoderStaticConfig enc{};
AudioHelper::call_function_sync(
    AudioHelper::FunctionId::SetEncoderStaticConfig,
    BROOKESIA_DESCRIBE_TO_JSON(enc).as_object());

AudioHelper::DecoderStaticConfig dec{};
AudioHelper::call_function_sync(
    AudioHelper::FunctionId::SetDecoderStaticConfig,
    BROOKESIA_DESCRIBE_TO_JSON(dec).as_object());
```

### 3. 绑定服务

```cpp
auto binding = service_manager.bind(AudioHelper::get_name().data());
BROOKESIA_CHECK_FALSE_EXIT(binding.is_valid(), "Failed to bind Audio service");
```

### 4. 取得存储文件路径（LittleFS 挂载点）

```cpp
static std::string get_audio_file_path(const std::string &filename) {
    auto [name, fs_iface] = hal::get_first_interface<hal::StorageFsIface>();
    for (const auto &info : fs_iface->get_all_info()) {
        if (info.fs_type == hal::StorageFsIface::FileSystemType::LittleFS) {
            return "file:/" + std::string(info.mount_point) + "/" + filename;
        }
    }
    return filename;
}
```

### 5. 播放单/多 URL 与播放控制

```cpp
// 订阅播放状态（schema: PlayState(String)）
auto conn = AudioHelper::subscribe_event(
    AudioHelper::EventId::PlayStateChanged,
    [](const std::string &, const std::string & state) {
        BROOKESIA_LOGI("Play state: %1%", state);
    });

// 单 URL（可带配置：是否打断、循环等）
AudioHelper::PlayUrlConfig cfg{ .interrupt = false, .loop_count = 5 };
AudioHelper::call_function_sync(
    AudioHelper::FunctionId::PlayUrl, get_audio_file_path("1.mp3"),
    BROOKESIA_DESCRIBE_TO_JSON(cfg).as_object());

// 多 URL（Array）
std::vector<std::string> urls = { get_audio_file_path("4.mp3"), get_audio_file_path("5.mp3") };
AudioHelper::call_function_sync(
    AudioHelper::FunctionId::PlayUrls,
    BROOKESIA_DESCRIBE_TO_JSON(urls).as_array());

// 播放控制（Action: String）
AudioHelper::call_function_sync(
    AudioHelper::FunctionId::PlayControl,
    BROOKESIA_DESCRIBE_TO_STR(AudioHelper::PlayControlAction::Pause));
AudioHelper::call_function_sync(
    AudioHelper::FunctionId::PlayControl,
    BROOKESIA_DESCRIBE_TO_STR(AudioHelper::PlayControlAction::Resume));
AudioHelper::call_function_sync(
    AudioHelper::FunctionId::PlayControl,
    BROOKESIA_DESCRIBE_TO_STR(AudioHelper::PlayControlAction::Stop));
```

### 6. 编解码回环（编码采集 → 解码播放）

```cpp
static AudioHelper::CodecGeneralConfig codec_general{
    .channels = 1, .sample_bits = 16, .sample_rate = 16000, .frame_duration = 60,
};

std::vector<uint8_t> recorded;
auto on_enc = [&](const std::string &, service::RawBuffer buf) {
    if (buf.data_ptr && buf.data_size > 0)
        recorded.insert(recorded.end(), buf.data_ptr, buf.data_ptr + buf.data_size);
};
auto enc_conn = AudioHelper::subscribe_event(AudioHelper::EventId::EncoderDataReady, on_enc);

auto timeout = service::helper::Timeout(1000);
AudioHelper::EncoderDynamicConfig enc_cfg{ .type = AudioHelper::CodecFormat::OPUS, .general = codec_general };
if (enc_cfg.type == AudioHelper::CodecFormat::OPUS) {
    enc_cfg.extra = AudioHelper::EncoderExtraConfigOpus{ .enable_vbr = false, .bitrate = 24000 };
}
AudioHelper::call_function_sync(
    AudioHelper::FunctionId::StartEncoder,
    BROOKESIA_DESCRIBE_TO_JSON(enc_cfg).as_object(), timeout);

boost::this_thread::sleep_for(boost::chrono::seconds(10));   // 采集 10s

AudioHelper::call_function_sync(AudioHelper::FunctionId::StopEncoder, timeout);

// 解码播放
AudioHelper::DecoderDynamicConfig dec_cfg{ .type = enc_cfg.type, .general = codec_general };
AudioHelper::call_function_sync(
    AudioHelper::FunctionId::StartDecoder,
    BROOKESIA_DESCRIBE_TO_JSON(dec_cfg).as_object(), timeout);

for (size_t off = 0; off < recorded.size(); off += 1024) {
    size_t n = std::min<size_t>(1024, recorded.size() - off);
    // FeedDecoderData 含 RawBuffer，不需要 task scheduler，故无需 timeout
    AudioHelper::call_function_sync(
        AudioHelper::FunctionId::FeedDecoderData, service::RawBuffer(recorded.data() + off, n));
}
AudioHelper::call_function_sync(AudioHelper::FunctionId::StopDecoder, timeout);
```

### 7. AFE（VAD + 唤醒词）

```cpp
AudioHelper::AFE_Config afe{
    .vad = AudioHelper::AFE_VAD_Config{},
    .wakenet = AudioHelper::AFE_WakeNetConfig{},   // 默认配置
};
AudioHelper::call_function_sync(
    AudioHelper::FunctionId::SetAFE_Config,
    BROOKESIA_DESCRIBE_TO_JSON(afe).as_object());

auto afe_conn = AudioHelper::subscribe_event(
    AudioHelper::EventId::AFE_EventHappened,
    [](const std::string &, const std::string & ev) {
        BROOKESIA_LOGI("AFE event: %1%", ev);
    });

// 查询唤醒词
auto ww = AudioHelper::call_function_sync<boost::json::array>(AudioHelper::FunctionId::GetAFE_WakeWords);

// 启动 encoder 才会驱动 AFE 处理
AudioHelper::EncoderDynamicConfig enc_cfg{ .type = AudioHelper::CodecFormat::PCM, .general = codec_general };
AudioHelper::call_function_sync(
    AudioHelper::FunctionId::StartEncoder,
    BROOKESIA_DESCRIBE_TO_JSON(enc_cfg).as_object(), timeout);
// ... 监听 AFE 事件 ...
AudioHelper::call_function_sync(AudioHelper::FunctionId::StopEncoder, timeout);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| bind 失败 | 未初始化 HAL 音频/存储设备 | 先 `hal::init_device(StorageDevice)` 与 `AudioDevice` |
| 播放无声 | menuconfig 未烧模型或音量为 0/静音 | 用 Device 服务 `SetAudioPlayerMute(false)` 并设音量 |
| PSRAM 任务崩溃 | 用 PSRAM 但未开 XIP | `player_task.stack_in_ext = true` |
| `FeedDecoderData` 超时 | 误传了 timeout | 该函数不需要 task scheduler，不要传 Timeout |
| OPUS 编码失败 | 未设 bitrate | `EncoderExtraConfigOpus{.bitrate=24000}` |
| G711A 不可用 | 在 ESP32-C5 上不支持 | C5 上跳过 `CodecFormat::G711A` |

## 参考

- `examples/service/audio/main/main.cpp` — set configs / play / control / codec / AFE 全套
- `docs/en/service/audio.rst` — Audio 服务接口契约
