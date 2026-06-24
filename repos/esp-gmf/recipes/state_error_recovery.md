# 状态机、ERROR 恢复与控制接口

> **适用摘要**: 理解 pipeline/task 状态机、各控制 API 的有效状态、ERROR 后的恢复步骤，以及 stop 超时、pause/resume、seek 的正确用法。

## 触发意图

- "流水线状态机"
- "ERROR 后怎么恢复"
- "stop 超时"
- "pause/seek"
- "pipeline 卡住"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考文档 | `docs/en/gmf-framework/gmf-core/gmf-core-overview.rst`、`gmf-core-pipeline.rst` |

## 分步说明

### 1. 状态机全景

```
NONE → INITIALIZED → OPENING → RUNNING → FINISHED   (is_done)
                         │        │ ↕ PAUSED
                         │        └→ STOPPED   (用户 stop)
                         └─────────└→ ERROR    (job FAIL)
   STOPPED / FINISHED / ERROR  ──reset──→  INITIALIZED
```

三个终态（STOPPED/FINISHED/ERROR）都可通过 `esp_gmf_pipeline_reset` 回到 INITIALIZED，即同一 pipeline 可反复 run/stop 而不重建。

### 2. 控制 API 的有效状态

| API | 有效状态 | 无效时返回 |
|---|---|---|
| `esp_gmf_pipeline_run` | INITIALIZED / STOPPED / FINISHED | `ESP_GMF_ERR_NOT_SUPPORT` |
| `esp_gmf_pipeline_stop` | RUNNING / PAUSED | — |
| `esp_gmf_pipeline_pause` | RUNNING | `ESP_GMF_ERR_NOT_SUPPORT` |
| `esp_gmf_pipeline_resume` | PAUSED | — |
| `esp_gmf_pipeline_reset` | 非 RUNNING | — |
| `esp_gmf_pipeline_seek` | PAUSED / STOPPED / FINISHED | — |

### 3. ERROR 恢复（最常见误区）

```c
// ❌ 错误 — ERROR 状态直接 run，返回 ESP_GMF_ERR_NOT_SUPPORT
esp_gmf_pipeline_run(pipe);

// ✅ 正确 — reset → loading_jobs → run
esp_gmf_pipeline_reset(pipe);          // 回到 INITIALIZED，清 job 列表
esp_gmf_pipeline_loading_jobs(pipe);   // reset 清空了 job，必须重新注册
// 重新设置输入 URI / 设备句柄等（如需要）
esp_gmf_pipeline_run(pipe);
```

错误处理路径（框架自动）：job FAIL → 移除未执行的 process job → 依次 close 已 open 的 element → pipeline 事件处理器收到 CHANGE_STATE=ERROR → 关闭头/尾 IO → 转发 ERROR 给用户回调。

### 4. stop 超时不是失败

```c
// 默认阻塞超时 DEFAULT_TASK_OPT_MAX_TIME_MS = 2000 ms
// ❌ 误区：把 ESP_GMF_ERR_TIMEOUT 当致命错误
esp_gmf_err_t r = esp_gmf_pipeline_stop(pipe);
if (r == ESP_GMF_ERR_TIMEOUT) panic();   // 错误

// ✅ 超时仅表示本次同步等待未完成，stop 仍在后台继续
esp_gmf_pipeline_stop(pipe);
// 等事件回调里的 ESP_GMF_EVENT_STATE_STOPPED 确认真正停止
```

stop 超时常见原因：element 在 process 里做长阻塞（网络/互斥）。处理：`esp_gmf_task_set_timeout(task, N)` 调大，或在 element 内查 abort 标志主动退出。

### 5. pause / resume / seek

```c
// pause 在下一个 job 边界生效（不会打断 element 处理中数据）
esp_gmf_pipeline_pause(pipe);   // 成功后查询应为 PAUSED
// ... 用户操作 ...
esp_gmf_pipeline_resume(pipe);  // 释放挂起信号量回 RUNNING

// seek 仅对支持独立帧解码的流（MP3/AAC/TS）有意义，且必须在非 RUNNING 状态
esp_gmf_pipeline_pause(pipe);
esp_gmf_pipeline_seek(pipe, offset_bytes);   // 调输入 IO 的 seek 接口跳转
```

> `pause` 在 job 边界生效。若 element 长时间不达边界，pause 返回 TIMEOUT，但 pause 请求仍会在下一个边界生效——立刻查状态可能仍是 RUNNING，属正常。

### 6. ABORT 与策略恢复

element 的 process 返回 `ESP_GMF_JOB_ERR_ABORT`（或数据层 `esp_gmf_db_abort`/`esp_gmf_io_abort`）会触发策略函数：

- 策略返回 `GMF_TASK_STRATEGY_ACTION_DEFAULT` → 走 STOPPED，需 reset 才能 run
- 策略返回 `GMF_TASK_STRATEGY_ACTION_RESET` → 回 INITIALIZED 自动恢复（如断网重连后继续播）

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| run 返回 NOT_SUPPORT | 状态不是可 run 的三个之一 | ERROR 后先 reset+loading_jobs |
| 一直停在 ERROR | 没调 reset | reset → loading_jobs → run |
| stop 永久不返回 | element process 长阻塞 | 调大 timeout 或 element 内查 abort 标志 |
| pause 后状态仍 RUNNING | element 未达 job 边界 | TIMEOUT 仅表示本次等待超时，请求仍会在下个边界生效 |
| reset 后 run 无反应 | 漏 loading_jobs | reset 会清 job 列表，必须重新 loading_jobs |

## 参考

- `docs/en/gmf-framework/gmf-core/gmf-core-overview.rst`（Lifecycle States、Event Types）
- `docs/en/gmf-framework/gmf-core/gmf-core-pipeline.rst`（Error Handling Path、Stop and Timeout、Recovery from ERROR）
