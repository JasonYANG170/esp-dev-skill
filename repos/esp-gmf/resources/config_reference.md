# ESP-GMF 配置参考

> 来自 `docs/`、各组件 `Kconfig`、`idf_component.yml` 与示例 `Kconfig.projbuild`。未列出的配置项视为不存在。

## 系统级前置

| 项 | 要求 |
|---|---|
| ESP-IDF | `>= v5.4.3 (release/v5.4)` / `>= v5.5.2 (release/v5.5)` / `>= v6.0` |
| 构建路径 | 不能含空格（ESP-IDF 限制） |
| 组件分发 | `idf_component.yml` 声明 `espressif/gmf_core`、`espressif/gmf_io` 等依赖，组件管理器自动拉取 |

## Task 默认配置（esp_gmf_task_cfg_t）

```c
DEFAULT_ESP_GMF_TASK_CONFIG()
// thread.stack = 4 * 1024
// thread.prio  = 5
// thread.core  = 0
// thread.stack_in_ext = false
// name = "gmf_task"
```

| 场景 | 推荐 task 栈 |
|---|---|
| 普通本地/Flash 播放 | 默认 4 KB（可按需上调） |
| HTTPS 播放 | ≥ 8 KB（TLS 握手，mbedtls_mpi_div_mpi） |
| 录音编码 AMR/AAC/OPUS | 40 KB，`stack_in_ext = true` 放 PSRAM |

控制 API 超时：默认 `DEFAULT_TASK_OPT_MAX_TIME_MS = 2000` ms，`esp_gmf_task_set_timeout(task, ms)` 调整。

## IO 配置（esp_gmf_io_cfg_t）

```c
typedef struct {
    struct { uint32_t stack; uint8_t prio; uint8_t core; } thread;  // stack==0 同步模式
    struct {
        uint32_t io_size;       // 每次底层 IO 读大小
        uint32_t buffer_size;   // databus 大小
        void    *read_filter;   // 自定义 payload 后处理（解密/去封装）
    } buffer_cfg;
    bool enable_speed_monitor;
} esp_gmf_io_cfg_t;
```

| 模式 | 条件 | 适用 |
|---|---|---|
| 同步 | `thread.stack == 0` 且无 `buffer_cfg` | codec_dev / embed_flash / i2s_pdm |
| 异步 | `thread.stack > 0` 且 `buffer_size > 0` | http（默认）、file（大块/抖动场景） |

io_http 默认异步：stack 6 KiB、prio 10、core 0、databus 20 KiB、io_block 3 KiB。

io_file 缓存：
- `cache_size`：`setvbuf` 用户缓冲；`<= 512` 关闭；以上按 512 对齐向上取整
- `cache_caps`：0 默认 `MALLOC_CAP_DMA`；esp32p4 等 PSRAM-DMA 芯片建议 `MALLOC_CAP_SPIRAM | MALLOC_CAP_DMA` 省 SRAM；esp32 用 `MALLOC_CAP_INTERNAL | MALLOC_CAP_8BIT`

## 四种 DataBus

| 实现 | 特点 |
|---|---|
| ringbuffer | 字节级，带拷贝 |
| fifo | 字节级队列 |
| block | 整块零拷贝，性能高、可配置性低 |
| pbuf | 链式缓冲 |

> port 绑定其一，element 代码看到的接口完全一致。block 型端口配合 `esp_gmf_cache` 做字节级读取（如解码器解析协议头）。

## FourCC 速查（esp_fourcc.h，节选）

| 类别 | 宏示例 |
|---|---|
| 音频编码 | `ESP_FOURCC_MP3` / `AAC` / `FLAC` / `OPUS` / `AMRNB` / `AMRWB` / `ALAC` / `ALAW` / `ULAW` / `ADPCM` / `PCM` / `PCM_S16/S24/S32` / `PCM_U08` / `VORBIS` / `SBC` / `LC3` / `G722` / `M4A` |
| 视频编码 | `ESP_FOURCC_H264` / `AVC1`（无起始码）/ `H265` / `H263` / `VP8` / `MJPG` |
| 容器 | `ESP_FOURCC_WAV` / `MP4` / `TS` / `M2TS` / `FLV` / `AVI` / `OGG` / `CAF` / `WEBM` |
| 图像 | `ESP_FOURCC_PNG` / `JPEG` / `GIF` / `WEBP` / `BMP` |
| 像素(RGB) | `ESP_FOURCC_RGB565→RGBL` / `RGB24` / `RGBA32` / `BGR24` ... |
| 像素(YUV) | `ESP_FOURCC_YUV420P` / `NV12` / `NV21` / `YUYV` / `UYVY` ... |
| 工具宏 | `ESP_FOURCC_TO_INT(a,b,c,d)` / `ESP_FOURCC_TO_STR(fourcc)` / `gmf_fourcc_to_str(fourcc, buf)` |

## URL 评分（IO 自动选择）

| 分数 | 含义 | 示例 |
|---|---|---|
| `ESP_GMF_IO_SCORE_NONE(0)` | 不支持 | — |
| `ESP_GMF_IO_SCORE_STANDARD(50)` | scheme 匹配 | `http://`→io_http，`file://`→io_file，`embed://`→io_embed_flash |
| `ESP_GMF_IO_SCORE_PERFECT(100)` | scheme+扩展/专用 IO | 自定义 HLS 抢占 HTTP |

## pipeline_audio_effects 示例 menuconfig（Kconfig.projbuild）

| 配置 | 默认 | 说明 |
|---|---|---|
| `CONFIG_GMF_AUDIO_EFFECT_INIT_MIXER` | y | 启用 mixer 演示 |
| `CONFIG_GMF_AUDIO_EFFECT_INIT_ALC` | y | ALC 演示 |
| `CONFIG_GMF_AUDIO_EFFECT_INIT_SONIC` | y | 变速变调 |
| `CONFIG_GMF_AUDIO_EFFECT_INIT_EQ` | y | EQ |
| `CONFIG_GMF_AUDIO_EFFECT_INIT_FADE` | y | 淡入淡出 |
| `CONFIG_GMF_AUDIO_EFFECT_INIT_DRC` | y | 单段动态范围 |
| `CONFIG_GMF_AUDIO_EFFECT_INIT_MBC` | y | 多段压缩 |

播放示例统一的 codec 采样率/通道/位深（来自 `gmf_loader` Kconfig）：
- `CONFIG_GMF_AUDIO_EFFECT_RATE_CVT_DEST_RATE`
- `CONFIG_GMF_AUDIO_EFFECT_CH_CVT_DEST_CH`
- `CONFIG_GMF_AUDIO_EFFECT_BIT_CVT_DEST_BITS`

## gmf_loader 覆盖范围（menuconfig 勾选）

| 类别 | 可选项 |
|---|---|
| IO reader | codec_dev RX / file / http / flash |
| IO writer | codec_dev TX / file / http |
| 音频解码 | MP3 / AAC / AMR-NB·WB / FLAC / WAV / M4A / TS / OGG / OPUS / G711 / PCM / ADPCM / LC3 / SBC / ALAC / VORBIS / G722 |
| 音频编码 | AAC / AMR-NB·WB / G711 / OPUS / ADPCM / PCM / ALAC / LC3 / SBC / G722 |
| 音频效果 | ALC / EQ / Mixer / Sonic / Fade / DRC / MBC / ch·bit·rate_cvt / ASRC / HOWL / interleave·deinterleave |
| 视频编解码 | H.264 / MJPEG SW·HW 编解码、PPA 加速 |
| 图像效果 | scale / rotate / crop / overlay / 帧率转换 / 颜色转换 |
| AI 音频 | WakeNet / AEC / 完整 AFE / VAD / NS / DOA |
| 容器（muxer） | TS / MP4 / FLV / WAV / CAF / OGG |

> 原则：仅启用实际需要的项，减小固件体积与内存占用。可通过 Kconfig 在不改 C 源码的前提下裁剪。

## esp_audio_simple_player 支持格式

解码：AAC, MP3, AMR, M4A, PCM, WAV, ADPCM, G711, OGG, VORBIS, OPUS, ALAC, FLAC, SBC, LC3, TS。
变换器：位深转换、通道转换、采样率转换（默认开），menuconfig 可裁剪。

## esp_player 支持

- 容器：WAV, MP4, M4A, TS, OGG, AVI, FLV, CAF；裸 ES：`.mp3/.aac/.flac/.amr`
- 音频解码：AAC, MP3, Vorbis, Opus, FLAC, AMR-NB/WB, G.711 A·μ-law, ALAC, ADPCM, SBC, LC3
- 视频解码：H.264, MJPEG
- 同步模式：AUDIO / VIDEO / SYSTEM / NONE
- URL scheme：file / http / https / fill / block
