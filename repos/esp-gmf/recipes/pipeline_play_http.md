# 播放 HTTP/HTTPS 网络音乐

> **适用摘要**: 构建「io_http → aud_dec → 效果链 → io_codec_dev」流水线播放网络音频，处理 HTTPS 的 TLS 握手栈、异步 IO 预取与控制超时。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-gmf/resources/`, source/examples in `repos/esp-gmf/`, and this recipe path `repos/esp-gmf/recipes/pipeline_play_http.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "播放网络音乐"
- "HTTP 音频播放"
- "HTTPS 播放"
- "流式音频"
- "io_http 怎么用"

## 前置条件

| 条件 | 要求 |
|---|---|
| 网络 | Wi-Fi 已连接（`esp_gmf_app_wifi_connect`） |
| 证书 | HTTPS 建议启用 `CONFIG_MBEDTLS_CERTIFICATE_BUNDLE` 或配置 `crt_bundle_attach` |
| 参考示例 | `gmf_examples/basic_examples/pipeline_play_http_music` |

## 分步说明

### 1. 连网与板级外设

```c
#include "esp_gmf_app_setup_peripheral.h"

playback_peripheral_init(&playback_handle);   // 同 embed 示例
esp_gmf_app_wifi_connect();
```

### 2. 建 pool 与 pipeline（头 IO 用 io_http）

```c
#define URI_HTTP "https://dl.espressif.com/dl/audio/gs-16b-2c-44100hz.mp3"

esp_gmf_pool_handle_t pool = NULL;
esp_gmf_pool_init(&pool);
gmf_loader_setup_io_default(pool);
gmf_loader_setup_audio_codec_default(pool);
gmf_loader_setup_audio_effects_default(pool);

const char *name[] = {"aud_dec", "aud_rate_cvt", "aud_ch_cvt", "aud_bit_cvt"};
esp_gmf_pipeline_handle_t pipe = NULL;
esp_gmf_pool_new_pipeline(pool, "io_http", name, 4, "io_codec_dev", &pipe);
esp_gmf_io_codec_dev_set_dev(ESP_GMF_PIPELINE_GET_OUT_INSTANCE(pipe), playback_handle);
esp_gmf_pipeline_set_in_uri(pipe, URI_HTTP);

// 从 URI 推断解码格式
esp_gmf_element_handle_t dec_el = NULL;
esp_gmf_pipeline_get_el_by_name(pipe, "aud_dec", &dec_el);
esp_gmf_info_sound_t info = {0};
esp_gmf_audio_helper_get_audio_type_by_uri(URI_HTTP, &info.format_id);
esp_gmf_audio_dec_reconfig_by_sound_info(dec_el, &info);
```

io_http 默认异步：stack 6 KiB、prio 10、core 0、数据总线 20 KiB、IO 块 3 KiB，无需手动配置即可吸收网络抖动。

### 3. 关键：HTTPS 必须把 task 栈调大并放宽超时

```c
esp_gmf_task_cfg_t cfg = DEFAULT_ESP_GMF_TASK_CONFIG();
cfg.name = "pipeline_task";
cfg.thread.stack = 8 * 1024;   // 默认 4 KiB 在 TLS 握手时会溢出（mbedtls_mpi_div_mpi）
esp_gmf_task_handle_t task = NULL;
esp_gmf_task_init(&cfg, &task);
esp_gmf_task_set_timeout(task, 20000);   // 网络阻塞需要更长等待
esp_gmf_pipeline_bind_task(pipe, task);
esp_gmf_pipeline_loading_jobs(pipe);
esp_gmf_pipeline_set_event(pipe, _pipeline_event, evt);

esp_gmf_pipeline_run(pipe);
```

### 4. （可选）自定义 HTTP 配置：证书、事件钩子

如需自签证书或自定义鉴权，单独构造 io_http 并注册进 pool 替换默认项：

```c
#include "esp_gmf_io_http.h"

http_io_cfg_t cfg = HTTP_STREAM_CFG_DEFAULT();
cfg.dir               = ESP_GMF_IO_DIR_READER;
cfg.crt_bundle_attach = esp_crt_bundle_attach;   // 需 CONFIG_MBEDTLS_CERTIFICATE_BUNDLE
cfg.event_handle      = my_http_hook;            // 在连接/请求/响应/结束/切轨时回调
cfg.user_data         = &app_ctx;

esp_gmf_io_handle_t io = NULL;
esp_gmf_io_http_init(&cfg, &io);
esp_gmf_pool_register_io(pool, io, NULL);        // 注册后 pool_new_pipeline 按名选用
```

事件枚举为 `http_stream_event_id_t`；回调返回正值会让 io_http 跳过默认读写，适合自定义鉴权/数据修改/分片上传。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| TLS 握手时复位/栈溢出 | task 栈默认 4 KiB 太小 | `cfg.thread.stack = 8 * 1024` |
| 控制接口返回 TIMEOUT | 网络阻塞超过 2 s 默认超时 | `esp_gmf_task_set_timeout(task, 20000)` |
| HTTPS 证书校验失败 | 未配置证书 | 启用 cert bundle 或设 `cert_pem` |
| 大文件播放卡顿 | 预取缓冲不够 | 增大 `http_io_cfg_t.io_cfg.buffer_size`（默认 20 KiB）与 `io_size`（默认 3 KiB） |
| acquire 读到的 valid_size < wanted | 网络分块/文件尾，属正常 | 以 `valid_size` 为准，`is_done==true` 时停止 |

## 参考

- `gmf_examples/basic_examples/pipeline_play_http_music/main/play_http_music.c`
- `docs/en/gmf-framework/gmf-elements/gmf-io.rst`（io_http 章节）
