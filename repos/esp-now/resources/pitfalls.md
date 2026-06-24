# ESP-NOW 常见陷阱汇总

> 汇总 SKILL.md 中 Critical Pitfalls 及 recipe 中的常见错误，便于快速查阅。所有 API/宏均来自仓库真实头文件。

## 初始化顺序

1. **espnow_init 必须在 Wi-Fi `esp_wifi_start()` 之后** — 否则 `espnow_init` 返回 ESP_FAIL。
2. **app_main 首步调用 `espnow_storage_init()`** — NVS 未初始化会导致 `espnow_storage_set/get`、绑定列表持久化、密钥存储全部失败。
3. **Wi-Fi 必须是 STA + RAM 存储 + PS_NONE** — 这是所有官方示例的统一做法；省电模式开启会导致 ESP-NOW 收发不稳定。
4. **`receive_enable.*` 默认多为 0** — 需要的功能要么在 `ESPNOW_INIT_CONFIG_DEFAULT()` 后手动设 `cfg.receive_enable.xxx=1`，要么由模块 API（如 `espnow_ctrl_responder_data`）内部 `set_config_for_data_type` 开启。

## 收发

5. **接收必须先 `espnow_set_config_for_data_type(type, true, cb)`** — 不注册回调永远收不到对应类型数据。
6. **单包数据 ≤ `ESPNOW_DATA_LEN`** — 明文 230 字节；开启加密后实际净荷为 `ESPNOW_SEC_PACKET_MAX_SIZE(=230-4-8)`。分配缓冲用对应宏。
7. **`frame_config` 传 NULL 用默认** — `ESPNOW_FRAME_CONFIG_DEFAULT()` 为 broadcast=true、retransmit_count=10。
8. **retransmit_count / forward_ttl ≤ 0x1f(31)** — 超限无意义，用 `ESPNOW_RETRANSMIT_MAX_COUNT`/`ESPNOW_FORWARD_MAX_COUNT`。

## 安全

9. **`sec_enable=1` 不等于自动加密** — 还需 `espnow_set_key`（发送加密）+ `espnow_set_dec_key`（接收解密），且 `frame_head.security=true` 才加密发送。
10. **只 set_key 不 set_dec_key** — 本端能加密发但无法解密收。两者必须成对。
11. **握手前数据是明文** — 用 `s_sec_flag` 在 `ESP_EVENT_ESPNOW_SEC_OK` 事件里置 true 后再加密发送。
12. **加密净荷更小** — 缓冲分配用 `ESPNOW_SEC_PACKET_MAX_SIZE`，不要用 `ESPNOW_DATA_LEN`。
13. **PoP 必须两端一致** — `CONFIG_APP_ESPNOW_SESSION_POP` 不匹配则握手失败。

## OTA

14. **initiator 必须先把固件 `esp_ota_write` 到本地 next update partition** — 直接 `espnow_ota_initiator_send` 而本地无固件，分发为空。
15. **分发回调内用 `esp_partition_read`** — 从已写入的升级分区读出，`src_offset` 为相对分区起始的偏移。
16. **SHA-256 用于断点续传比对** — 用 `esp_partition_get_sha256` 取分区真实哈希（前 16 字节 `ESPNOW_OTA_HASH_LEN`）。
17. **scan/result 内存必须成对释放** — `espnow_ota_initiator_scan_result_free` + `espnow_ota_initiator_result_free(&result)` + `ESP_FREE(dest)`。
18. **responder 必须 `espnow_ota_responder_start`** — 否则 initiator 扫描不到、分发无效。
19. **版本检查** — `skip_version_check=true` 跳过；否则 responder 比对版本决定是否升级。

## 设备控制

20. **responder 必须先进入绑定窗口** — `espnow_ctrl_responder_bind(wait_ms, rssi, cb)`，否则 initiator 发 bind 无效。
21. **控制值是 `uint32_t`** — 回调签名固定 `uint32_t responder_value`，不是 bool/int 二选一；0/1/2 等自定义编码。
22. **bindlist 上限 32** — `ESPNOW_BIND_LIST_MAX_SIZE`，超限用 `espnow_ctrl_responder_clear_bindlist`。
23. **bind 帧 RSSI 阈值** — `espnow_ctrl_responder_bind` 的 `rssi` 参数过低会被忽略，靠近设备或调低阈值（如 -70）。
24. **set_bindlist 才持久化** — `remove_bindlist` 删除；重启后 `get_bindlist` 应能读回 `set` 的项。

## 配网

25. **responder 回调返回值决定是否下发** — initiator 信息校验回调返回 `ESP_OK` 才回发 Wi-Fi 配置。
26. **packed 结构勿直接取成员地址** — 先 `memcpy` 到本地 `wifi_config_t` 再 `esp_wifi_set_config`（见 provisioning 示例注释）。
27. **自定义数据需填 `custom_size`** — `espnow_prov_initiator_t.custom_size` + `custom_data[]`。

## 低功耗

28. **硬币电池发包前 light sleep 充电** — `CONFIG_ESPNOW_LIGHT_SLEEP=y`，发包前 `set_light_sleep(SEND_GAP_TIME)`，否则复位。
29. **`esp_wifi_force_wakeup_acquire/release` 配对** — 睡前 release，醒后 acquire。
30. **`esp_now_set_wake_window(0)`** — 关闭唤醒窗口省电（见 coin_cell switch app_main）。
31. **`send_max_timeout = portMAX_DELAY`** — 硬币电池场景阻塞直到发送成功。

## 内存与工具

32. **`ESP_MALLOC/CALLOC/REALLOC/FREE` 带调试记录** — `CONFIG_ESPNOW_MEM_DEBUG=y` 时 `espnow_mem_print_record()` 可查泄漏。
33. **`ESP_REALLOC_RETRY` 会阻塞** — 堆不足时一直重试，确认请求合理。
34. **NVS key ≤ 15 字符，value ≤ 1984 字节** — 见 `espnow_storage_set` 文档。
35. **回调内勿做重活** — 数据接收在 espnow 任务上下文执行，复杂处理移交应用任务（Kconfig 注释明确）。
36. **重启判定** — `espnow_reboot_is_exception(true)` 判断异常重启并擦 coredump；连续重启达 `CONFIG_ESPNOW_REBOOT_UNBROKEN_FALLBACK_COUNT`(默认 30) 触发版本回滚。

## 构建/烧录

37. **首次或换芯片必须 `idf.py set-target`** — 否则 sdkconfig 不匹配。
38. **首次烧录建议 `idf.py erase_flash`** — 残留 NVS/分区可能导致绑定列表或密钥异常。
39. **组件依赖用 add-dependency** — 不要手动复制 `src/`，交由组件管理器下载到 `managed_components/`。
