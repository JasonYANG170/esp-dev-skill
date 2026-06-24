# 播放列表管理：SD 卡扫描(sdcard_scan) + 多存储后端 playlist

> **适用摘要**: 用 `sdcard_scan()` 扫描 SD 卡音频文件并经回调存入 playlist，再用 `playlist_operator_handle_t` 句柄做 next / prev / choose(id) / current，由 `playlist_handle_t` 管理多个列表。四种存储后端：SD 卡（`sdcard_list`）、DRAM（`dram_list`）、NVS Flash（`flash_list`）、DATA_UNDEFINED 分区（`partition_list`）。播放时用 `audio_element_set_uri()` 把列表给出的 URL 喂给 fatfs_stream 并 reset pipeline 切歌。

## 触发意图

- "播放列表 / playlist / 下一首 / 上一首"
- "扫描 SD 卡里的歌 / sdcard_scan"
- "按序号点歌 / choose by id"
- "列表存到 NVS / 断电不丢"
- "pipeline_sdcard_mp3_control 那个例子"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/player/pipeline_sdcard_mp3_control/main/play_sdcard_mp3_control_example.c`、`examples/cli` |
| 头文件 | `components/playlist/include/playlist.h`、`sdcard_scan.h`、`sdcard_list.h`、`dram_list.h`、`flash_list.h`、`partition_list.h` |
| 硬件 | SD 卡（`audio_board_sdcard_init`）；partition_list 还需在分区表加两个 `subtype=0x06/0x07` 的分区 |
| 文档 | `docs/en/api-reference/playlist/index.rst` |

## 分步说明

### 1. 四种存储后端对照

| 后端 | create 函数 | 特点 |
|---|---|---|
| SD 卡 | `sdcard_list_create(&h)` | 列表本身存 SD 卡，断电保留，依赖 fatfs |
| DRAM | `dram_list_create(&h)` | 存内存，掉电丢失，最快 |
| NVS Flash | `flash_list_create(&h)` | 存 NVS 分区，断电保留，容量受 NVS 限制 |
| Partition | `partition_list_create(&h)` | 存 `DATA_UNDEFINED` 分区，需加 subtype 0x06/0x07 两个分区 |

> 四者都返回同一个抽象 `playlist_operator_handle_t`，操作接口由 `playlist_operation_t` 函数指针表统一（见 `playlist.h`）。

### 2. 创建列表 + 扫描 SD 卡（最常用：SD 卡后端）

```c
#include "playlist.h"
#include "sdcard_scan.h"
#include "sdcard_list.h"
#include "board.h"

static playlist_operator_handle_t sdcard_list_handle;

// sdcard_scan 每扫到一个匹配文件就调一次，把 URL 存进列表
void sdcard_url_save_cb(void *user_data, char *url)
{
    playlist_operator_handle_t h = (playlist_operator_handle_t)user_data;
    esp_err_t ret = sdcard_list_save(h, url);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Fail to save url to playlist");
    }
}

// 挂载 SD 卡
audio_board_sdcard_init(set, SD_MODE_1_LINE);

// 建列表 + 扫描（depth=0 表只扫 /sdcard 根目录；过滤 .mp3）
sdcard_list_create(&sdcard_list_handle);
sdcard_scan(sdcard_url_save_cb, "/sdcard", 0,
            (const char *[]){"mp3"}, 1, sdcard_list_handle);
sdcard_list_show(sdcard_list_handle);   // 打印所有 URL
```

> `sdcard_scan(cb, path, depth, file_extension[], filter_num, user_data)`：`depth=0` 只扫当前目录，`depth=1` 扫一层子目录，依此类推；`file_extension` 不含点号。

### 3. 用 playlist 管理多个列表（可选）

`playlist_handle_t` 是「列表的列表」，可把 sdcard_list / dram_list 等多个实例挂进去，用 `list_id` 切换：

```c
playlist_handle_t plist = playlist_create();
playlist_add(plist, sdcard_list_handle, 0);   // list_id = 0
// 也可再 add 一个 dram_list，list_id = 1
playlist_checkout_by_id(plist, 0);            // 切到 id=0

int n = playlist_get_current_list_url_num(plist);
char *url = NULL;
playlist_get_current_list_url(plist, &url);   // 当前 URL
playlist_next(plist, 1, &url);                // 下一首
playlist_prev(plist, 1, &url);                // 上一首
playlist_choose(plist, 3, &url);              // 跳到 id=3
```

> 多数简单场景直接用 `sdcard_list_next/prev/choose/current` 操作单个句柄即可，不必建 `playlist_handle_t`。

### 4. 播放：取 URL → set_uri → reset pipeline → run

```c
#include "fatfs_stream.h"
#include "mp3_decoder.h"
#include "i2s_stream.h"
#include "filter_resample.h"

// 取第一首 URL 并设给 fatfs reader
char *url = NULL;
sdcard_list_current(sdcard_list_handle, &url);

fatfs_stream_cfg_t fatfs_cfg = FATFS_STREAM_CFG_DEFAULT();
fatfs_cfg.type = AUDIO_STREAM_READER;
audio_element_handle_t fatfs_reader = fatfs_stream_init(&fatfs_cfg);
audio_element_set_uri(fatfs_reader, url);

// mp3 -> resample -> i2s (略，参考 play_sdcard_fatfs.md)
// ...
audio_pipeline_run(pipeline);
```

### 5. 一首播完自动切下一首（关键模式）

监听 i2s writer 的 `AEL_STATE_FINISHED`，取下一首 URL 重置 pipeline（**不要 terminate**，那样会重建所有 element 任务、切歌很慢）：

```c
if (msg.source == (void *)i2s_stream_writer
    && msg.cmd == AEL_MSG_CMD_REPORT_STATUS) {
    if (audio_element_get_state(i2s_stream_writer) == AEL_STATE_FINISHED) {
        char *url = NULL;
        sdcard_list_next(sdcard_list_handle, 1, &url);
        audio_element_set_uri(fatfs_reader, url);
        audio_pipeline_reset_ringbuffer(pipeline);
        audio_pipeline_reset_elements(pipeline);
        audio_pipeline_change_state(pipeline, AEL_STATE_INIT);
        audio_pipeline_run(pipeline);
    }
}
```

> 这套 `set_uri → reset_ringbuffer → reset_elements → change_state(INIT) → run` 是 ADF 官方推荐的「热切歌」模式，出自 `pipeline_sdcard_mp3_control` 注释。

### 6. 按键控制（next / prev / 选号）

```c
// 在 input_key_service 回调里
if ((int)msg.data == get_input_set_id()) {        // SET = 下一首
    char *url = NULL;
    sdcard_list_next(sdcard_list_handle, 1, &url);
    audio_element_set_uri(fatfs_reader, url);
    audio_pipeline_reset_ringbuffer(pipeline);
    audio_pipeline_reset_elements(pipeline);
    audio_pipeline_change_state(pipeline, AEL_STATE_INIT);
    audio_pipeline_run(pipeline);
}
```

### 7. 用 NVS / Partition 后端（断电保留）

```c
// NVS（容量小，适合存少量 URL）
playlist_operator_handle_t flash_list_handle;
flash_list_create(&flash_list_handle);
flash_list_save(flash_list_handle, "http://example.com/a.mp3");

// DATA_UNDEFINED 分区（需先在 partitions.csv 加 subtype 0x06/0x07 两个分区）
playlist_operator_handle_t part_list_handle;
partition_list_create(&part_list_handle);
partition_list_save(part_list_handle, "/sdcard/b.mp3");
```

> `partition_list_create` 头文件注释明确：**必须先加两个 subtype 为 0x06 和 0x07 的分区**，否则失败。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 扫不到文件 | `depth` 太小或扩展名写错 | `depth` 按目录层级调；扩展名不带点（`"mp3"` 不是 `".mp3"`） |
| 切歌卡顿 / 有杂音 | 切歌时 terminate 了 pipeline | 用 `set_uri + reset_ringbuffer + reset_elements + change_state(INIT) + run` 热切 |
| `sdcard_list_next` 返回的 URL 重复 | 没用 `step` | `next(h, 1, &url)` 步进 1；`choose(h, id, &url)` 按绝对 id |
| partition_list_create 失败 | 缺分区 | 分区表加 subtype 0x06 和 0x07 两个分区 |
| 多个列表 id 冲突 | 不同列表用了相同 list_id | `playlist_add` 的 `list_id` 必须全局唯一（跨 handle 也唯一） |
| URL 乱码 | fatfs 长文件名 / 中文 | `CONFIG_FATFS_LFN_NONE` 改为支持 LFN |
| 第一首不播 | 未 `set_uri` 就 run | `audio_element_set_uri(fatfs_reader, url)` 必须在 run 前 |

## 参考项目

- `examples/player/pipeline_sdcard_mp3_control/main/play_sdcard_mp3_control_example.c` — SD 卡扫描 + 列表 + 按键 + 热切歌完整实现
- `examples/cli` — CLI 命令操作 playlist
- `docs/en/api-reference/playlist/index.rst` — 四种存储后端说明
- `components/playlist/include/playlist.h`、`sdcard_scan.h`、`sdcard_list.h`、`dram_list.h`、`flash_list.h`、`partition_list.h` — 真实 API
