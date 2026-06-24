# 无缝循环播放

> **适用摘要**: 用 `esp_gmf_task_set_strategy_func` 注册策略函数，在播放结束（FINISH/ABORT）时返回 `GMF_TASK_STRATEGY_ACTION_RESET` 自动续播下一曲，配合 IO reload 实现「不重建 pipeline 的无缝切歌」。

## 触发意图

- "无缝循环播放"
- "playlist 自动续播"
- "切歌不中断"
- "strategy_func"
- "loop play"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `gmf_examples/basic_examples/pipeline_loop_play_no_gap` |
| 参考文档 | `docs/en/gmf-framework/gmf-core/gmf-core-pipeline.rst`（Task Strategy and Abort Flow） |

## 分步说明

### 1. 准备播放列表与策略上下文

```c
#define PLAYBACK_DURATION_MS  20000

static const char *play_urls[] = {
    "/sdcard/track1.mp3",
    "/sdcard/track2.mp3",
    "/sdcard/track3.mp3",
};

typedef struct {
    bool       stop_loop;
    int        play_index;
    int        file_count;
    const char **file_path;
    esp_gmf_io_handle_t io;   // 头 IO 句柄（pipe->in）
} pipeline_strategy_ctx_t;
```

### 2. 实现 strategy 函数（FINISH 触发时换 URI 并 RESET）

策略函数在两类触发点被调用：`GMF_TASK_STRATEGY_TYPE_FINISH`（所有 job DONE）与 `GMF_TASK_STRATEGY_TYPE_ABORT`（job ABORT）。返回值决定后续走向。**注意：策略函数内禁止调用任何 pipeline/task 控制 API**，否则触发超时。

```c
static int pipeline_strategy_finish_func(esp_gmf_task_handle_t task,
                                         gmf_task_strategy_type_t type,
                                         void *ctx)
{
    pipeline_strategy_ctx_t *s = (pipeline_strategy_ctx_t *)ctx;
    if (s->stop_loop) {
        return GMF_TASK_STRATEGY_ACTION_STOP;   // 用户要停，走正常 STOP
    }
    if (type == GMF_TASK_STRATEGY_TYPE_FINISH) {
        // 当前曲结束，切下一曲（循环回绕）
        s->play_index = (s->play_index + 1) % s->file_count;
        esp_gmf_io_set_uri(s->io, s->file_path[s->play_index]);
        // RESET 动作：恢复到 INITIALIZED，不重建 pipeline，task 继续调度
        return GMF_TASK_STRATEGY_ACTION_RESET;
    }
    return GMF_TASK_STRATEGY_ACTION_DEFAULT;
}

// 需要停止循环时（如延迟一段时间后）
static void _pipeline_set_finish_stop_strategy(pipeline_strategy_ctx_t *s) {
    s->stop_loop = true;
}
```

### 3. 注册策略、绑定 task、loading_jobs、run

```c
pipeline_strategy_ctx_t strategy_ctx = {
    .stop_loop   = false,
    .play_index  = 0,
    .file_count  = sizeof(play_urls) / sizeof(char *),
    .file_path   = play_urls,
    .io          = pipe->in,    // 头 IO
};
esp_gmf_task_set_strategy_func(work_task, pipeline_strategy_finish_func, &strategy_ctx);

esp_gmf_pipeline_bind_task(pipe, work_task);
esp_gmf_pipeline_loading_jobs(pipe);
esp_gmf_pipeline_set_event(pipe, _pipeline_event, evt);

esp_gmf_pipeline_run(pipe);

// 播 20 s 后改为停止策略，等当前曲自然结束即退出
vTaskDelay(PLAYBACK_DURATION_MS / portTICK_PERIOD_MS);
_pipeline_set_finish_stop_strategy(&strategy_ctx);

xEventGroupWaitBits(evt, PIPELINE_BLOCK_BIT, pdTRUE, pdFALSE, portMAX_DELAY);
esp_gmf_pipeline_stop(pipe);
```

### 4. （可选）同主机分段用 io_reload 复用连接

对 HLS 分段等「同一 host 连续多段」，用 `esp_gmf_io_reload` 复用 HTTP 连接，省去重新握手：

```c
// 在事件回调或策略函数中（注意不能调 pipeline 控制 API，但 io_reload 是 IO 级接口）
esp_gmf_io_set_uri(io, next_segment_uri);
esp_gmf_io_reload(io);   // 复用连接与资源，比 close+open 快
```

> `reload` 必须在上一次读完成后调用。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| strategy 里调 run/stop 触发超时 | 策略函数禁止调控制 API | 只返回动作枚举；要等外部事件用信号量阻塞 |
| 切歌后有断音 | 每次重建 pipeline | 用 `GMF_TASK_STRATEGY_ACTION_RESET`，不重建 |
| 无法停止 | 只设了 RESET 策略无退出条件 | 加 `stop_loop` 标志，置位后返回 `ACTION_STOP` |
| RESET 后 IO 不前进 | `io_done` 后未 clear | 切歌流程里 `io_done` → 等 pipeline → `set_uri` → `clear_done` |

## 参考

- `gmf_examples/basic_examples/pipeline_loop_play_no_gap/main/play_music_without_gap.c`
- `docs/en/gmf-framework/gmf-core/gmf-core-pipeline.rst`（Task Strategy and Abort Flow）
