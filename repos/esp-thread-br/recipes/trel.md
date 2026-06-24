# TREL (Thread Radio Encapsulation Link)

> **适用摘要**: 让具备 Wi-Fi 但无 802.15.4 的设备（如 ESP32-S3）通过 TREL 经 Wi-Fi 直接参与 Thread 网络，与 Thread CLI 设备互通。

## 触发意图
- "TREL"
- "Thread over Wi-Fi"
- "Thread Radio Encapsulation Link"
- "无 15.4 设备加入 Thread"

## 前置条件

| 条件 | 要求 |
|---|---|
| 设备 | ESP Thread BR + Thread CLI 设备 + TREL Wi-Fi 设备（如 ESP32-S3） |
| 网络 | BR 与 TREL Wi-Fi 设备同 Wi-Fi；CLI 加入 BR 的 Thread 网络 |
| 固件 | TREL Wi-Fi 设备刷 ESP-IDF `examples/openthread/ot_trel` |
| 参考 | `docs/en/codelab/trel.rst`、`docs/en/codelab/basic_setup.rst` |

## 分步说明

### 1. 在 BR 启用 TREL

```bash
idf.py menuconfig
# Component config -> OpenThread -> OpenThread -> Thread Core Features
#   -> Thread Trel Radio Link -> Enable Thread Radio Encapsulation Link (TREL)
# 即 CONFIG_OPENTHREAD_RADIO_TREL=y
```

> 启用后 `app_main` 中 eventfd 数量需 +1：
```c
size_t max_eventfd = 3;
#if CONFIG_OPENTHREAD_RADIO_TREL
    max_eventfd++;   // TREL reception needs an eventfd
#endif
```

### 2. 起网（BR + CLI + TREL 设备）

BR 形成网络并取 dataset：
```
# BR
> ot dataset active -x
```

Thread CLI 与 TREL Wi-Fi 设备均设同一 dataset 后 `ifconfig up` + `thread start`，最终都应是 `router` 或 `child`：
```
> ot state
router
```

### 3. 验证 TREL 连通（互 ping mleid）

TREL Wi-Fi 设备：
```
> ot ipaddr mleid
fd14:c8eb:d14c:5fbe:bd5e:16de:3183:694a
```

Thread CLI 设备：
```
> ot ipaddr mleid
fd14:c8eb:d14c:5fbe:b57e:1e02:a532:26d1
> ot ping fd14:c8eb:d14c:5fbe:bd5e:16de:3183:694a
16 bytes from ... icmp_seq=1 hlim=255 time=122ms
1 packets transmitted, 1 packets received. Packet loss = 0.0%.
```

反向同理。`hlim=255` 表示同链路（TREL 把 Wi-Fi 当 Thread 链路层）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| TREL 设备入网失败 | BR 与 TREL 设备 Wi-Fi 不同 | 同一 Wi-Fi，且 BR 已 `CONFIG_OPENTHREAD_RADIO_TREL=y` |
| eventfd 不足崩溃 | TREL 未计入 eventfd | `max_eventfd` 在 TREL 时 +1 |
| ping mleid 不通 | 取错地址类型 | 必须 ping `mleid`（mesh-local endpoint），不是 link-local |
| BR 同时跑 15.4 与 TREL | 配置混乱 | BR 可双链路；TREL-only 设备无需 15.4 |

## 参考
- `docs/en/codelab/trel.rst`（3.6）
- `docs/en/codelab/basic_setup.rst`（TREL 配置项位置）
- `examples/basic_thread_border_router/main/esp_ot_br.c`（eventfd 累加注释）
