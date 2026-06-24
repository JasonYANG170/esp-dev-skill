# AGENTS.md — Supplementary Agent Guide

> 核心规则、状态机、陷阱、配方索引、执行工作流均在 `SKILL.md`。
> 本文件只覆盖 `SKILL.md` 未涉及的约定与工具链指引，不重复内容。

## Project Context

**语言**: C · **目标**: ESP32 系列 SoC（ESP32 / ESP32-S3 / ESP32-P4 / ESP32-C3 等） · **工具链/构建**: ESP-IDF（`idf.py`），版本 `>= v5.4.3 (release/v5.4)` / `>= v5.5.2 (release/v5.5)` / `>= v6.0` · **组件分发**: ESP Component Manager（`idf_component.yml` 声明依赖，构建时自动拉取）

## Code Generation Conventions

### File Naming
- 源文件 `*.c`，头文件 `*.h`，公共头集中在各组件 `include/`
- element 头：`esp_gmf_<element>.h`（如 `esp_gmf_audio_dec.h`、`esp_gmf_eq.h`）
- IO 头：`esp_gmf_io_<type>.h`（`esp_gmf_io_file.h`、`esp_gmf_io_http.h`、`esp_gmf_io_embed_flash.h`、`esp_gmf_io_codec_dev.h`、`esp_gmf_io_i2s_pdm.h`）
- 示例主文件以场景命名：`play_embed_music.c`、`play_http_music.c`、`play_record_sdcard.c` 等

### Include Pattern（手工编排 pipeline 时）
```c
#include "esp_gmf_element.h"
#include "esp_gmf_pipeline.h"
#include "esp_gmf_pool.h"
#include "esp_gmf_audio_dec.h"
#include "esp_gmf_audio_helper.h"
#include "esp_gmf_io_codec_dev.h"     // 或 io_file / io_http / io_embed_flash
#include "gmf_loader_setup_defaults.h" // gmf_loader_setup_* / teardown_* 声明
#include "esp_board_manager_includes.h"
#include "esp_codec_dev.h"
```

### Standard Project Structure（来自 `gmf_examples`）
```
my_gmf_app/
├── main/
│   ├── CMakeLists.txt
│   ├── idf_component.yml          # 声明 espressif/gmf_examples 等依赖
│   ├── Kconfig.projbuild          # 可选：示例级 menuconfig 选项
│   └── <app>.c                    # app_main()
├── CMakeLists.txt                 # ESP-IDF 顶层
├── sdkconfig.defaults             # 可选：默认配置（PSRAM、板子等）
└── partitions.csv                 # 可选：分区表
```

### Canonical app_main Pattern（pool + pipeline）
```c
void app_main(void)
{
    // 1. 外设：板级 codec/SD/Wi-Fi（经 esp_board_manager / gmf_app_utils）
    playback_peripheral_init(&playback_handle);

    // 2. 建 pool，用 gmf_loader 批量注册 IO/codec/effects
    esp_gmf_pool_handle_t pool = NULL;
    esp_gmf_pool_init(&pool);
    gmf_loader_setup_io_default(pool);
    gmf_loader_setup_audio_codec_default(pool);
    gmf_loader_setup_audio_effects_default(pool);
    ESP_GMF_POOL_SHOW_ITEMS(pool);

    // 3. 按名建 pipeline（头/尾 IO + element 名数组）
    const char *name[] = {"aud_dec", "aud_rate_cvt", "aud_ch_cvt", "aud_bit_cvt"};
    esp_gmf_pipeline_handle_t pipe = NULL;
    esp_gmf_pool_new_pipeline(pool, "io_file", name, 4, "io_codec_dev", &pipe);
    esp_gmf_io_codec_dev_set_dev(ESP_GMF_PIPELINE_GET_OUT_INSTANCE(pipe), playback_handle);
    esp_gmf_pipeline_set_in_uri(pipe, "/sdcard/test.mp3");

    // 4. 配解码格式（用 helper 从 URI 推断 FourCC）
    esp_gmf_element_handle_t dec_el = NULL;
    esp_gmf_pipeline_get_el_by_name(pipe, "aud_dec", &dec_el);
    esp_gmf_info_sound_t info = {0};
    esp_gmf_audio_helper_get_audio_type_by_uri("/sdcard/test.mp3", &info.format_id);
    esp_gmf_audio_dec_reconfig_by_sound_info(dec_el, &info);

    // 5. task + bind + loading_jobs + event（顺序固定）
    esp_gmf_task_cfg_t cfg = DEFAULT_ESP_GMF_TASK_CONFIG();
    cfg.name = "player";
    esp_gmf_task_handle_t task = NULL;
    esp_gmf_task_init(&cfg, &task);
    esp_gmf_pipeline_bind_task(pipe, task);
    esp_gmf_pipeline_loading_jobs(pipe);
    EventGroupHandle_t evt = xEventGroupCreate();
    esp_gmf_pipeline_set_event(pipe, _pipeline_event, evt);

    // 6. run → 等待 FINISHED/ERROR/STOPPED → stop → 销毁
    esp_gmf_pipeline_run(pipe);
    xEventGroupWaitBits(evt, BIT(0), pdTRUE, pdFALSE, portMAX_DELAY);
    esp_gmf_pipeline_stop(pipe);

    esp_gmf_task_deinit(task);
    esp_gmf_pipeline_destroy(pipe);
    gmf_loader_teardown_audio_effects_default(pool);
    gmf_loader_teardown_audio_codec_default(pool);
    gmf_loader_teardown_io_default(pool);
    esp_gmf_pool_deinit(pool);
}
```

### 事件回调模板
```c
static esp_gmf_err_t _pipeline_event(esp_gmf_event_pkt_t *event, void *ctx)
{
    ESP_LOGI(TAG, "EVT el:%s type:%x sub:%s",
             OBJ_GET_TAG(event->from), event->type,
             esp_gmf_event_get_state_str(event->sub));
    if (event->sub == ESP_GMF_EVENT_STATE_STOPPED
        || event->sub == ESP_GMF_EVENT_STATE_FINISHED
        || event->sub == ESP_GMF_EVENT_STATE_ERROR) {
        xEventGroupSetBits((EventGroupHandle_t)ctx, BIT(0));
    }
    return ESP_GMF_ERR_OK;
}
```

## Build Workflow

1. **激活 ESP-IDF 环境**，确认 `idf.py --version` 为受支持版本
2. **选板**：`pip install esp-bmgr-assist` → `idf.py bmgr -l` → `idf.py bmgr -b <board_name|index>`
3. **配置**：`idf.py menuconfig`（裁剪解码器/效果/IO，调整 PSRAM、栈）
4. **构建**：`idf.py build`
5. **烧录监视**：`idf.py -p PORT flash monitor`（默认波特 460800）
6. **退出监视**：`Ctrl+]`

> 路径中不能有空格（ESP-IDF 构建系统限制）。

## Code Generation Checklist

- [ ] 依赖在 `main/idf_component.yml` 声明（`espressif/gmf_examples` 或具体组件）
- [ ] 调用顺序：`pool_init → new_pipeline → set_in/out_uri → task_init → bind_task → loading_jobs → set_event → run`
- [ ] element 名数组与 pool 注册 tag 一致（`aud_dec`/`aud_rate_cvt`/`aud_ch_cvt`/`aud_bit_cvt`/`aud_enc`/`aud_muxer`/`aud_eq`...）
- [ ] 头/尾 IO tag 正确（`io_file`/`io_http`/`io_embed_flash`/`io_codec_dev`/`io_i2s_pdm`）
- [ ] 显式配置 `aud_dec` 的 `dec_type` 或用 helper reconfig（避免误判）
- [ ] 录音编码：task 栈足够（AMR/AAC 40 KB，可放 PSRAM）；bitrate 按格式合法值
- [ ] HTTPS 播放：task 栈 ≥ 8 KB，`esp_gmf_task_set_timeout(task, 20000)`
- [ ] 事件回调处理 STOPPED/FINISHED/ERROR 三个终态，避免死等
- [ ] 销毁顺序：`task_deinit → pipeline_destroy → gmf_loader_teardown_*（逆序）→ pool_deinit → 板级 deinit`
- [ ] 自定义 element：派生类首成员为基类；`acquire/release` 严格成对；`is_done` 正确传播

## Do Not Modify

- `resources/` — 文档来源，不要改
- `SKILL.md` frontmatter — 技能元数据
- 仓库源码（`esp-gmf/` 下各组件源文件）— 只引用，不改写

## Notes on Grounding

- 本技能所有函数名、结构体、宏、配置项、Kconfig 符号、文件路径均来自 `docs/` 与组件源码的真实定义。仓库未覆盖的 API **一律不写**。
- element tag（如 `aud_dec`）来自 `gmf_loader` 默认注册；自定义 element 自行决定 tag。
- 板级初始化经 `esp_board_manager`（外部仓库）与 `gmf_app_utils`；本技能不展开其内部 API。
