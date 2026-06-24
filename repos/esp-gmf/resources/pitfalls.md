# ESP-GMF 陷阱汇总

> 把 `SKILL.md` 的「Critical Pitfalls」展开，每条给出症状、根因与正确做法。

## 1. 构建顺序：必须 bind_task → loading_jobs → set_event → run

- 症状：run 后无声音、无任何 element 被调度。
- 根因：`loading_jobs` 把 element 的 open/process 注册进 task job 列表，跳过则 job 列表为空。
- 正确：
  ```c
  esp_gmf_pipeline_bind_task(pipe, task);
  esp_gmf_pipeline_loading_jobs(pipe);
  esp_gmf_pipeline_set_event(pipe, cb, ctx);
  esp_gmf_pipeline_run(pipe);
  ```

## 2. reset 后必须重新 loading_jobs

- 症状：ERROR/STOPPED 后 reset 再 run 无反应。
- 根因：`reset` 清空 job 列表与 element/port 状态。
- 正确：
  ```c
  esp_gmf_pipeline_reset(pipe);
  esp_gmf_pipeline_loading_jobs(pipe);
  esp_gmf_pipeline_run(pipe);
  ```

## 3. ERROR 状态不能直接 run

- 症状：run 返回 `ESP_GMF_ERR_NOT_SUPPORT`。
- 根因：ERROR 是终态，必须先 reset 回 INITIALIZED。
- 正确：reset → loading_jobs → run。

## 4. acquire/release 必须成对（含错误分支）

- 症状：跑一段时间后 port 泄漏、缓冲耗尽、卡死。
- 根因：process 里 acquire 后出错直接 return，未 release。
- 正确：每个错误分支先 release 已 acquire 的 payload；返回前 release_out 再 release_in。

## 5. is_done 必须传播，否则 pipeline 不结束

- 症状：源 IO 读到文件尾，pipeline 却一直 RUNNING。
- 根因：自定义 element 没把 `in_load->is_done` 赋给 `out_load->is_done`，也没返回 `ESP_GMF_JOB_ERR_DONE`。
- 正确：
  ```c
  out_load->is_done = in_load->is_done;
  /* ... release ... */
  return in_load->is_done ? ESP_GMF_JOB_ERR_DONE : ESP_GMF_JOB_ERR_OK;
  ```

## 6. 依赖型 element 迟迟不 open

- 症状：`dependency=true` 的 element（rate_cvt/asrc）始终不启动。
- 根因：上游没调 `esp_gmf_element_notify_snd_info` 上报格式信息，pipeline 没注册其 open/process job。
- 正确：上游 open/process 解析出采样率后用 `GMF_AUDIO_UPDATE_SND_INFO(self, rate, bits, ch)`；或应用层 `esp_gmf_pipeline_report_info(pipe, ESP_GMF_INFO_SOUND, &info, sizeof(info))` 主动上报（录音场景必备）。

## 7. 解码器格式未显式配置或依赖误判

- 症状：aud_dec 解码失败或音质异常。
- 根因：`dec_type` 未设、`use_frame_dec=true` 又没指定格式；自动检测对某些流误判。
- 正确：显式设 `dec_type`，或用 `esp_gmf_audio_helper_get_audio_type_by_uri` 推断 FourCC 后 `esp_gmf_audio_dec_reconfig_by_sound_info`。

## 8. HTTPS 播放栈溢出

- 症状：HTTPS 播放在 TLS 握手时复位/`mbedtls_mpi_div_mpi` 栈溢出。
- 根因：默认 task 栈 4 KB 不够 TLS。
- 正确：`cfg.thread.stack = 8 * 1024`；网络阻塞还需 `esp_gmf_task_set_timeout(task, 20000)`。

## 9. element 名 / IO tag 拼写不符 pool 注册

- 症状：`esp_gmf_pool_new_pipeline` 找不到模板失败。
- 根因：名字数组与 pool 注册 tag 不一致。
- 正确：用 `ESP_GMF_POOL_SHOW_ITEMS(pool)` 确认真实 tag（如 `aud_dec`/`aud_rate_cvt`/`aud_ch_cvt`/`aud_bit_cvt`/`aud_enc`/`aud_muxer`/`aud_eq`/`io_file`/`io_http`/`io_embed_flash`/`io_codec_dev`/`io_i2s_pdm`）。

## 10. stop 超时被误判为失败

- 症状：把 `ESP_GMF_ERR_TIMEOUT` 当致命错误，重复 stop 致状态混乱。
- 根因：stop 超时仅表示本次同步等待未完成，stop 仍在后台继续。
- 正确：忽略超时返回值，靠事件回调的 `ESP_GMF_EVENT_STATE_STOPPED` 确认；或在 element 内查 abort 标志主动退出长阻塞。

## 11. 切歌后无声音

- 症状：`esp_gmf_io_done` 后换 URI，下游 IO 不前进。
- 根因：done 会挂起异步 IO 的 task，必须 `clear_done` 才能继续。
- 正确：
  ```c
  esp_gmf_io_done(io);
  /* 等 pipeline 处理完残留 */
  esp_gmf_io_set_uri(io, next);
  esp_gmf_io_clear_done(io);
  ```

## 12. embed_flash URL 缺下划线索引

- 症状：报 "No _ in file name"。
- 根因：URL 必须为 `embed://<group>/<index>_<name>.<ext>`，下划线分隔索引与文件名。
- 正确：用 `mk_flash_embed_tone.py` 生成的标准 URI，或手动 `embed://tone/0_startup.mp3`。

## 13. 录音编码栈不足 / bitrate 非法

- 症状：AMR/AAC/OPUS 编码运行时栈溢出或编码失败。
- 根因：默认 4 KB 栈；AMR bitrate 用了任意数值。
- 正确：task 栈 40 KB + `stack_in_ext=true`；AMRWB 用 `ESP_AMRWB_ENC_BITRATE_MD885`，AMRNB 用 `ESP_AMRNB_ENC_BITRATE_MR122`。

## 14. 录音方向未 report_info

- 症状：aud_enc 未初始化即被调度，录音失败。
- 根因：录音无解码器解析文件头，编码器不知目标格式。
- 正确：`esp_gmf_audio_enc_reconfig_by_sound_info` + `esp_gmf_pipeline_report_info(pipe, ESP_GMF_INFO_SOUND, &info, sizeof(info))`。

## 15. 策略函数内调控制 API 死锁

- 症状：策略函数里调 `esp_gmf_pipeline_stop/run`，触发超时错误。
- 根因：策略函数在 task 调度上下文，调控制 API 会自等自己。
- 正确：策略函数只返回动作枚举（`GMF_TASK_STRATEGY_ACTION_DEFAULT/_RESET/_STOP`）；要等外部事件用信号量阻塞。

## 16. codec_dev IO 采样率与效果链不一致

- 症状：扬声器无声或杂音。
- 根因：`esp_codec_dev_open` 的采样率/通道/位深与 `aud_rate_cvt/aud_ch_cvt/aud_bit_cvt` 目标不符。
- 正确：codec_dev 用 menuconfig 的 `CONFIG_GMF_AUDIO_EFFECT_*_DEST_*` 配置，让效果链输出对齐硬件。

## 17. pause 后立刻查状态仍 RUNNING

- 症状：`pause` 后查询显示 RUNNING。
- 根因：pause 在 job 边界生效；element 长时间不达边界时 `pause` 返回 TIMEOUT，但请求仍会在下个边界生效。
- 正确：TIMEOUT 不代表取消；稍后再查应为 PAUSED。

## 18. aud_muxer 无输出

- 症状：muxer element 不产生容器数据。
- 根因：`enc → muxer` 顺序颠倒，或 muxer `codec` 字段与 aud_enc 实际输出不符，或流式模式却设了 FILE。
- 正确：保持 enc 在 muxer 前；`codec` 匹配编码格式；流式输出用 `OUTPUT_STREAMING`。

## 19. aud_sonic speed=2 输出非半长被误判为 bug

- 症状：time-stretch 后输出长度不固定。
- 根因：time-stretch 是非整数分帧，element 按速度比计算并保留累积误差。
- 正确：下游（codec_dev IO）必须按 `valid_size` 写硬件，不能假设固定长度。

## 20. 自定义 element 强转失败

- 症状：派生类与基类互转异常。
- 根因：基类不是结构体首成员。
- 正确：派生类第一字段必须是 `esp_gmf_audio_element_t parent`（或 video/pic 对应基类）。
