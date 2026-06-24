# ESP-WHO 任务状态机与数据流

> 所有状态位与流程均来自 `components/who_task/who_task.cpp`、`who_frame_cap_node.cpp`、`who_detect.cpp`、`who_recognition.cpp`、`who_qrcode.cpp` 的真实实现。

---

## 1. `WhoTaskBase` 生命周期状态机

```
        run() ────────────────────────────────┐
          │                                    │
          ▼                                    │
   [TASK_STOPPED]  ──run()成功──▶  [RUNNING]   │
          ▲                          │   │     │
          │                  pause() │   │ stop()
          │                          ▼   │     │
          │                     [TASK_PAUSED]  │
          │                          │   │     │
          │                 resume() │   │ stop()
          │                          ▼   │     │
          │                       [RUNNING]    │
          └──────── wait_for_stopped ◀────────┘
```

控制位（由调用方置位，任���内消费）：

| 调用 | 置位 | 任务内响应 | 任务内置回 |
|---|---|---|---|
| `stop_async()` | `TASK_STOP` | 收到后 `break` | `TASK_STOPPED` |
| `pause_async()` | `TASK_PAUSE` | 收到后置 `TASK_PAUSED`，再等 `TASK_RESUME`/`TASK_STOP` | — |
| `resume()` | `TASK_RESUME`（清 `TASK_PAUSED`） | 唤醒继续循环 | — |

> `stop()` = `stop_async()` + `wait_for_stopped()` + `cleanup_for_stopped()`；`pause()` 类似。

---

## 2. 帧采集节点链数据流

```
WhoCam ──(cam_fb_get)──▶ WhoFetchNode ──queue(len=1)──▶ [WhoDecodeNode] ──▶ [WhoPPAResizeNode] ──▶ 下游
                              │                                                                  │
                              └─ RingBuf<cam_fb_t*> (len = cam_fb_count-2)                        │
                                  满 → 给订阅者置 NEW_FRAME  ◀──────────────────────────────────┘
```

- 相邻节点之间是**长度为 1** 的 `QueueHandle_t`（`WhoFrameCap::add_node` 自动创建）。
- `out_queue_overwrite=true` 时用 `xQueueOverwrite`（只保留最新），`false` 时用 `xQueueSend`（阻塞，要求下游及时消费）。
- 节点 ringbuf 满后才向订阅者（如 `WhoDetect`/`WhoQRCode`）发 `NEW_FRAME`。
- `cam_fb_peek(-1)` 取 ringbuf 中最新帧；`WhoFrameLCDDisp` 默认 `peek_index=0` 取最旧帧（与检测结果对齐）。

### 2.1 `WhoFetchNode` 内部循环

```
loop:
  out_fb = cam->cam_fb_get()            // 阻塞取新帧
  if ringbuf full: cam->cam_fb_return(旧帧)  // 回收
  push(out_fb)
  if ringbuf full: 向订阅者置 NEW_FRAME
  overwrite/send 到 out_queue
```

### 2.2 `WhoDecodeNode` / `WhoPPAResizeNode`

- Decode：`CONFIG_SOC_JPEG_CODEC_SUPPORTED` ? `hw_decode_jpeg` : `sw_decode_jpeg`；失败返回 `nullptr`（节点丢弃该帧）。
- PPAResize：`dl::image::resize_ppa(*fb, dst_img, ppa_srm_handle)`，仅 `CONFIG_SOC_PPA_SUPPORTED`。

---

## 3. `WhoDetect` 任务循环

```
loop:
  等 NEW_FRAME | TASK_PAUSE | TASK_STOP
  if TASK_STOP: break
  if TASK_PAUSE: 置 TASK_PAUSED, 等 RESUME/STOP
  fb = frame_cap_node->cam_fb_peek()        // 最新帧
  img = static_cast<img_t>(*fb)
  res = model->run(img)                     // ESP-DL 推理
  if 设了 rescale_params: rescale_detect_result(res)   // 还原坐标
  if result_cb: result_cb({res, timestamp, img})
  if m_interval: vTaskDelayUntil(interval)  // set_fps 生效
```

---

## 4. `WhoRecognitionCore` 事件驱动（非循环轮询）

`WhoRecognitionCore` 通过三个事件位由按键触发：

```
           按键 PLAY / LVGL "recognize"        按键 UP / "enroll"        按键 DOWN / "delete"
                    │ RECOGNIZE                       │ ENROLL                  │ DELETE
                    ▼                                 ▼                         │
   动态替换 WhoDetect 的 detect_result_cb：           │                         │
     recognize(img,res) → 命中输出 "id: x, sim: y"   enroll(img,res)           delete_last_feat()
     未命中输出 "who?"                                成功输出 "id: n enrolled"  成功输出 "id: n deleted"
                                                     失败输出 "Failed to ..."   失败输出 "Failed to ..."
   执行完后把 detect_result_cb 还原回原回调            │                         │
```

> 关键：识别/注册**不是独立推理**，而是“挂载一次性回调”到下游 `WhoDetect`；当下游检测到帧时触发。删除则直接调用 `m_recognizer->delete_last_feat()`。

按键事件位映射（`who_recognition_button.cpp`）：
- `m_btn_user_data[0]` → `RECOGNIZE`
- `m_btn_user_data[1]` → `ENROLL`
- `m_btn_user_data[2]` → `DELETE`

物理按键 BSP 映射：

| 板子 | recognize | enroll | delete |
|---|---|---|---|
| `BSP_BOARD_ESP32_S3_EYE` | `BSP_BUTTON_PLAY` | `BSP_BUTTON_UP` | `BSP_BUTTON_DOWN` |
| `BSP_BOARD_ESP32_S3_KORVO_2` | `BSP_BUTTON_PLAY` | `BSP_BUTTON_VOLUP` | `BSP_BUTTON_VOLDOWN` |
| 其它（fallback） | index 0 | index 1 | index 2 |

触发方式统一为 `iot_button_register_cb(..., BUTTON_SINGLE_CLICK, ...)`。P4 触摸屏走 LVGL `LV_EVENT_CLICKED`。

---

## 5. `WhoQRCode` 任务循环

```
loop:
  等 NEW_FRAME | TASK_PAUSE | TASK_STOP
  fb = frame_cap_node->cam_fb_peek()
  data = quirc_begin()                       // 取灰度缓冲
  ImageTransformer.set_src_img(*fb).set_dst_img({data,w,h,GRAY}).transform()
  quirc_end()
  for i in quirc_count():
    quirc_extract → quirc_decode
    if err == QUIRC_ERROR_DATA_ECC: quirc_flip + 再 decode
    if 成功: result_cb(payload); break        // 只处理一个码
```

quirc 尺寸：S3 = `BSP_LCD_H_RES × BSP_LCD_V_RES`；P4 = `BSP_LCD_H_RES/2 × BSP_LCD_V_RES/2`。

---

## 6. `WhoApp` 一键启动顺序（以 `WhoDetectAppLCD::run()` 为例）

```
1. WhoYield2Idle::get_instance()->run()           // 启动空闲监控（栈深 2048）
2. for node in frame_cap->get_all_nodes():
       node->run(4096, 2, 0)                       // 采集节点，核0
3. m_lcd_disp->run(2560, 2, 0)                     // LCD 显示，核0
4. m_detect->run(4096, 2, 1)                       // 检测，核1
```

> 三类任务栈深/核绑定来自各 App 的 `run()` 实现，是**已调好的组合**，改时需同步改 `set_fps`/`ringbuf_len`/`fb_count`。
