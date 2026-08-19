---
name: esp-now-skill
description: >-
  AI Skill for Espressif ESP-NOW connectionless Wi-Fi protocol component development. Used when
  users need to create, modify, or debug ESP-NOW based firmware on ESP32-series chips, including
  data broadcast/unicast, device control & binding, security handshake, OTA, Wi-Fi provisioning,
  wireless debug, time synchronization, and coin-cell low-power switching.
  Trigger words: "ESP-NOW", "esp-now", "Espressif", "乐鑫", "ESP32", "ESP32-C3", "ESP32-S3",
  "ESP32-C2", "ESP32-C6", "无连接通信", "广播", "组网", "绑定", "OTA", "配网", "provisioning"
license: Apache-2.0
metadata:
  author: Community
  version: "1.1.0"
---

# esp-now-skill

面向乐鑫 ESP-NOW 无连接 Wi-Fi 通信组件（application-level ESP-NOW）的 AI 技能。本组件在 ESP-IDF 原生 `esp_now` 协议之上封装了数据收发、设备控制与绑定、安全握手、批量 OTA、Wi-Fi 配网、无线调试与时间同步等高级能力。本技能提供场景化 recipes、完整 API 速查、Kconfig 配置参考与常见陷阱，所有函数名、结构体、宏与示例代码均取自 `espressif/esp-now` 仓库真实源码。

## Core Principles

1. **绝不臆造 API** — 任何函数名/结构体/宏必须能在 `resources/api_reference.md` 或仓库头文件中查到；查不到即视为不存在。
2. **初始化顺序固定** — `espnow_storage_init()` → Wi-Fi 初始化（`esp_wifi_start`）→ `espnow_init(&config)`。顺序颠倒会导致 Wi-Fi 与 ESP-NOW 状态不一致。
3. **Wi-Fi 必须先以 STA 模式启动** — ESP-NOW 依赖底层 Wi-Fi；示例统一调用 `esp_wifi_set_mode(WIFI_MODE_STA)`、`esp_wifi_set_storage(WIFI_STORAGE_RAM)`、`esp_wifi_set_ps(WIFI_PS_NONE)`、`esp_wifi_start()` 后再 `espnow_init`。
4. **数据类型驱动收发** — 收发以 `espnow_data_type_t`（ACK/FORWARD/CONTROL_BIND/OTA_DATA/SECURITY/…）划分管道；发送用 `espnow_send(type, ...)`，接收用 `espnow_set_config_for_data_type(type, enable, cb)` 注册回调。
5. **payload 受限** — 单包用户数据最大 `ESPNOW_PAYLOAD_LEN (230)`；开启安全加密后实际净荷为 `ESPNOW_SEC_PACKET_MAX_SIZE`；`espnow_send` 的 `size` 不得超过 `ESPNOW_DATA_LEN`。
6. **安全加密需两步** — `espnow_config_t.sec_enable=1` 仅打开能力；真正加解密需先 `espnow_sec_*` 握手或 `espnow_set_key`/`espnow_set_dec_key` 写入派生密钥，且帧头 `frame_head.security=true` 才会加密发送。
7. **initiator/responder 角色分离** — control/security/ota/provisioning 都按 initiator（发起方，如开关/传感器）与 responder（响应方，如灯/插座）划分；同一设备可同时担任两种角色。
8. **OTA initiator 需先写入本地再分发** — initiator 先通过 HTTP 把固件下载并 `esp_ota_write` 到本地 `next update partition`，再以回调 `esp_partition_read` 分发给 responder，并基于 SHA-256 比对断点续传。
9. **结果内存必须释放** — `espnow_ota_initiator_result_free`、`espnow_sec_initiator_result_free`、`*_scan_result_free` 必须成对调用，否则内存泄漏。
10. **硬币电池方案需 light sleep** — 发包耗电大，`CONFIG_ESPNOW_LIGHT_SLEEP` 开启后组件在发包前进入 light sleep 给电容充电；配合 `esp_wifi_force_wakeup_acquire/release` 与 `esp_now_set_wake_window` 使用。
11. **事件统一走 esp_event** — 所有 control/ota/sec/timesync 事件挂在 `ESP_EVENT_ESPNOW` base 上，用 `esp_event_handler_register(ESP_EVENT_ESPNOW, ESP_EVENT_ANY_ID, cb, ...)` 捕获。
12. **回调中勿做重活** — 数据接收回调在 espnow 任务上下文执行；复杂处理（JSON、加密、写 flash）应通过队列移交应用任务（Kconfig 注释明确说明）。
13. **集成多特性时安全握手必须先于加密业务** — 在同一固件中组合 security + provisioning/control 时，加密的配网/控制数据必须等 `espnow_get_key()` 返回 ESP_OK（即 initiator 握手已派发 app key）后才能收发；`examples/solution` 的 responder 配网任务在握手前显式轮询 `espnow_get_key` 阻塞，否则数据走不通。

## When to Use

**Applicable:**
- 在 ESP-IDF 工程中集成 ESP-NOW 组件，实现点对点/广播/组播数据收发
- 实现设备控制（initiator 绑定/解绑 + responder 接收控制数据，如开关控灯）
- 批量固件 OTA 升级（initiator 分发，responder 接收，断点续传、版本回滚）
- 设备间安全握手（ECDH + AES-CCM）并加密 ESP-NOW 数据
- 通过 ESP-NOW 进行 Wi-Fi 配网（provisioning，已联网设备把 SSID/密码传给新设备）
- 无线调试：远程抓取设备日志、下发调试命令、console 交互
- 节点间时间同步（无需联网，initiator 广播权威时间）
- 硬币电池低功耗开关（light sleep + 控制发包）

**Not applicable:**
- 原生 ESP-IDF `esp_now_*` 低层 API 问题（请参考 ESP-IDF 文档，非本组件）
- 非 ESP32 系列芯片（如 ESP8266 RTOS SDK，需用对应组件）
- 蓝牙 BLE Mesh / Wi-Fi TCP/UDP socket 通信
- PCB 硬件设计与射频匹配

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先阅读对应 recipe**，其中包含完整调用链、分步说明、真实代码与常见错误。

### 基础收发

| recipe | scenario |
|---|---|
| `recipes/get_started_send_recv.md` | ESP-NOW 入门：Wi-Fi + storage + espnow_init 初始化，广播发送与回调接收（参考 get-started 示例） |
| `recipes/unicast_and_group.md` | 单播（`espnow_add_peer`）与分组（`espnow_add_group`/`espnow_set_group`）控制 |

### 设备控制

| recipe | scenario |
|---|---|
| `recipes/control_initiator.md` | initiator 侧：按键触发绑定/解绑与控制数据发送（参考 control 示例） |
| `recipes/control_responder.md` | responder 侧：进入绑定窗口、注册控制回调、维护绑定列表并执行动作 |
| `recipes/coin_cell_switch.md` | 硬币电池低功耗开关：light sleep、power lock、状态持久化（参考 coin_cell_demo/switch） |

### 安全 / OTA / 配网

| recipe | scenario |
|---|---|
| `recipes/security_handshake.md` | 安全握手：initiator 分发 app key、responder 启动握手、加解密收发（参考 security 示例） |
| `recipes/ota_batch_upgrade.md` | 批量 OTA：initiator 下载+分发固件、responder 启动升级、断点续传（参考 ota 示例） |
| `recipes/provisioning_wifi.md` | ESP-NOW Wi-Fi 配网：responder 广播 beacon、initiator 请求并应用 SSID/密码（参考 provisioning 示例） |

### 调试 / 时间 / 存储

| recipe | scenario |
|---|---|
| `recipes/wireless_debug.md` | 无线调试：日志等级配置、flash 日志读取、console 与命��注册（参考 wireless_debug 示例） |
| `recipes/time_sync.md` | 节点间时间同步：initiator 广播权威时间、responder 调整本地时间 |
| `recipes/storage_utils.md` | NVS 存储封装、内存调试宏与重启计数工具（espnow_storage / espnow_mem / espnow_utils） |

### 综合方案（产品级集成）

| recipe | scenario |
|---|---|
| `recipes/solution_integrated_firmware.md` | 多特性集成固件：单二进制按 `CONFIG_APP_ESPNOW_INITIATOR/RESPONDER` 条件编译集成 Wi-Fi 配网 + ESP-NOW 配网 + 控制 + 调试 + OTA + 安全 + 时间同步；单按键复用、共享 LED 状态、安全先于加密业务的关键约束（参考 solution 示例） |

---

## 关键配置参考

### ESP-NOW 数据类型 `espnow_data_type_t`（来自 `espnow.h`）

| 枚举值 | 用途 |
|---|---|
| `ESPNOW_DATA_TYPE_ACK` | 可靠传输的 ACK |
| `ESPNOW_DATA_TYPE_FORWARD` | 转发分组 |
| `ESPNOW_DATA_TYPE_GROUP` | 分组类型包 |
| `ESPNOW_DATA_TYPE_PROV` | Wi-Fi 配网包 |
| `ESPNOW_DATA_TYPE_CONTROL_BIND` | 绑定/解绑包 |
| `ESPNOW_DATA_TYPE_CONTROL_DATA` | 控制数据包 |
| `ESPNOW_DATA_TYPE_OTA_STATUS` | 批量升级状态包 |
| `ESPNOW_DATA_TYPE_OTA_DATA` | 批量升级数据包 |
| `ESPNOW_DATA_TYPE_DEBUG_LOG` | 调试日志包 |
| `ESPNOW_DATA_TYPE_DEBUG_COMMAND` | 调试命令包 |
| `ESPNOW_DATA_TYPE_DATA` | 用户自定义数据 |
| `ESPNOW_DATA_TYPE_SECURITY_STATUS` | 安全状态包 |
| `ESPNOW_DATA_TYPE_SECURITY` | 安全握手包 |
| `ESPNOW_DATA_TYPE_SECURITY_DATA` | 安全数据包 |
| `ESPNOW_DATA_TYPE_TIMESYNC` | 时间同步包 |

### 角色（来自 User_Guide.md）

| 角色 | 数据流向 | 典型设备 |
|---|---|---|
| Initiator（发起方） | 发出控制/配网/OTA/时间 | 开关、传感器、LCD 屏 |
| Responder（响应方） | 接收并执行 | 灯、插座、智能应用 |

> 同一设备可同时具备两种角色。

### 关键事件 base 与 offset

| 定义 | 值 |
|---|---|
| `ESP_EVENT_ESPNOW_PROV_BASE` | 0x100 |
| `ESP_EVENT_ESPNOW_CTRL_BASE` | 0x200 |
| `ESP_EVENT_ESPNOW_OTA_BASE` | 0x300 |
| `ESP_EVENT_ESPNOW_DEBUG_BASE` | 0x400 |
| `ESP_EVENT_ESPNOW_TIMESYNC_BASE` | 0x500 |
| `ESP_EVENT_ESPNOW_SEC_OK` / `_FAIL` | 0x600 / 0x601 |

完整事件 ID 与 Kconfig 选项见 `resources/config_reference.md`。

---

## Critical Pitfalls (Must Read)

下列错误最为常见，违反任一条都会导致固件不工作。

### 1. espnow_init 必须在 Wi-Fi start 之后

```c
// ❌ WRONG — 先 espnow_init，Wi-Fi 还没起来
espnow_config_t cfg = ESPNOW_INIT_CONFIG_DEFAULT();
espnow_init(&cfg);
esp_wifi_start();

// ✅ CORRECT — storage → wifi init+start → espnow_init
espnow_storage_init();
app_wifi_init();   // 内部 esp_wifi_start()
espnow_config_t cfg = ESPNOW_INIT_CONFIG_DEFAULT();
espnow_init(&cfg);
```

### 2. Wi-Fi 必须是 STA 模式并关闭省电（示例统一做法）

```c
// ❌ WRONG — 用默认 power save，ESP-NOW 收发不稳
esp_wifi_init(&cfg);
esp_wifi_start();

// ✅ CORRECT — STA + RAM 存储 + PS_NONE（见各示例 app_wifi_init）
ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
ESP_ERROR_CHECK(esp_wifi_set_storage(WIFI_STORAGE_RAM));
ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_NONE));
ESP_ERROR_CHECK(esp_wifi_start());
```

### 3. 接收必须先用 espnow_set_config_for_data_type 注册回调

```c
// ❌ WRONG — 只发送，没有注册接收回调，永远收不到数据
espnow_init(&cfg);

// ✅ CORRECT — 显式 enable + 回调
espnow_set_config_for_data_type(ESPNOW_DATA_TYPE_DATA, true, app_recv_cb);
// 回调签名: esp_err_t cb(uint8_t *src_addr, void *data, size_t size, wifi_pkt_rx_ctrl_t *rx_ctrl)
```

### 4. 数据长度不能超过 ESPNOW_DATA_LEN

```c
// ❌ WRONG — size 超过 230(明文)/加密后净荷更小
uint8_t buf[512] = {0};
espnow_send(ESPNOW_DATA_TYPE_DATA, addr, buf, 512, &fh, portMAX_DELAY);

// ✅ CORRECT — 用 ESPNOW_DATA_LEN 限制
uint8_t *data = ESP_CALLOC(1, ESPNOW_DATA_LEN);
size_t size = uart_read_bytes(PORT, data, ESPNOW_DATA_LEN, pdMS_TO_TICKS(10));
espnow_send(ESPNOW_DATA_TYPE_DATA, ESPNOW_ADDR_BROADCAST, data, size, &fh, portMAX_DELAY);
```

### 5. 安全加密：仅 sec_enable=1 不会自动加密

```c
// ❌ WRONG — 以为 sec_enable=1 数据就加密了
espnow_config_t cfg = ESPNOW_INIT_CONFIG_DEFAULT();
cfg.sec_enable = 1;
espnow_init(&cfg);
// 此时 frame_head.security 默认 0，仍是明文！

// ✅ CORRECT — 握手成功后把 frame_head.security 置 true 再发送
espnow_frame_head_t fh = { .broadcast = true, .retransmit_count = 10, .security = s_sec_flag };
espnow_send(ESPNOW_DATA_TYPE_DATA, ESPNOW_ADDR_BROADCAST, data, size, &fh, portMAX_DELAY);
// s_sec_flag 在 ESP_EVENT_ESPNOW_SEC_OK 事件里置 true
```

### 6. OTA initiator 必须先把固件写到本地分区

```c
// ❌ WRONG — 直接调用 espnow_ota_initiator_send，但本地还没有固件
espnow_ota_initiator_send(addrs, num, sha, 0, my_cb, &res);

// ✅ CORRECT — 先 HTTP 下载 + esp_ota_begin/write/end 写到 next update partition
const esp_partition_t *p = esp_ota_get_next_update_partition(NULL);
esp_ota_begin(p, total_size, &ota_handle);
/* 循环 esp_http_client_read → esp_ota_write */
esp_ota_end(ota_handle);
esp_partition_get_sha256(p, sha_256);
espnow_ota_initiator_send(addrs, num, sha_256, total_size, app_ota_initiator_data_cb, &res);
// 回调里用 esp_partition_read(p, src_offset, dst, size) 读出分发
```

### 7. scan/result 内存必须成对释放

```c
// ❌ WRONG — 只 scan，不 free
espnow_ota_initiator_scan(&info_list, &num, pdMS_TO_TICKS(3000));
// ...
espnow_ota_initiator_send(..., &result);
// 没有 *_result_free，内存泄漏

// ✅ CORRECT
espnow_ota_initiator_scan(&info_list, &num, pdMS_TO_TICKS(3000));
/* 拷贝 dest_addr_list */
espnow_ota_initiator_scan_result_free();
espnow_ota_initiator_send(dest_addr_list, num, sha, size, cb, &result);
/* 使用 result */
espnow_ota_initiator_result_free(&result);
ESP_FREE(dest_addr_list);
```

### 8. 控制绑定要先进入 responder 绑定窗口

```c
// ❌ WRONG — responder 没调用 espnow_ctrl_responder_bind，initiator 发 bind 也无效
espnow_ctrl_initiator_bind(ESPNOW_ATTRIBUTE_KEY_1, true); // initiator 侧

// ✅ CORRECT — responder 进入绑定窗口（带超时与 RSSI 阈值）
ESP_ERROR_CHECK(espnow_ctrl_responder_bind(30 * 1000, -55, NULL));
espnow_ctrl_responder_data(app_ctrl_data_cb);
// initiator 双击触发 espnow_ctrl_initiator_bind(KEY_1, true)
// 绑定结果通过 ESP_EVENT_ESPNOW_CTRL_BIND 事件回调拿到
```

### 9. control 数据值是 uint32，不是 bool/int 二选一

```c
// ❌ WRONG — 以为 responder 回调里 value 类型不定
void cb(..., bool on) { /* 编译不过 */ }

// ✅ CORRECT — 回调签名固定为 uint32_t responder_value
void app_responder_ctrl_data_cb(espnow_attribute_t initiator_attribute,
                                espnow_attribute_t responder_attribute,
                                uint32_t status) {
    if (status) app_led_set_color(255,255,255);
    else        app_led_set_color(0,0,0);
}
// 发送端: espnow_ctrl_initiator_send(ESPNOW_ATTRIBUTE_KEY_1, ESPNOW_ATTRIBUTE_POWER, status);
```

### 10. provisioning 回调返回值决定 responder 是否回发 Wi-Fi 配置

```c
// ❌ WRONG — initiator 信息回调总返回 ESP_OK，却不做校验
esp_err_t cb(uint8_t *src, void *data, size_t size, wifi_pkt_rx_ctrl_t *rx) {
    return ESP_OK; // responder 会无条件把 Wi-Fi 配置发给任意 initiator
}

// ✅ CORRECT — 在 responder 的 initiator 信息回调里校验 product_id/secret，
//             返回 ESP_OK 才回发 Wi-Fi 配置（见 espnow_prov_responder_start 第 4 参 cb 文档）
esp_err_t app_espnow_prov_recv_cb(...) {
    espnow_prov_initiator_t *info = data;
    if (strcmp(info->product_id, "my_product") != 0) return ESP_FAIL;
    return ESP_OK;
}
```

### 11. 硬币电池方案发包前要 light sleep 充电

```c
// ❌ WRONG — 上电立即发包，电容未充满，发包失败/复位
espnow_ctrl_initiator_send(ESPNOW_ATTRIBUTE_KEY_1, ESPNOW_ATTRIBUTE_POWER, 2);

// ✅ CORRECT — 配置 CONFIG_ESPNOW_LIGHT_SLEEP=y，发包前 set_light_sleep(SEND_GAP_TIME)
set_light_sleep(SEND_GAP_TIME); // ~30ms 充电
espnow_ctrl_initiator_send(ESPNOW_ATTRIBUTE_KEY_1, ESPNOW_ATTRIBUTE_POWER, status);
// 且 app_main 里: esp_now_set_wake_window(0);
```

### 12. 安全密钥需同时 set_key 与 set_dec_key

```c
// ❌ WRONG — 只调 espnow_set_key，本端能加密发但无法解密收
if (espnow_get_key(key_info) != ESP_OK) esp_fill_random(key_info, APP_KEY_LEN);
espnow_set_key(key_info);

// ✅ CORRECT — set_key（发送加密）与 set_dec_key（接收解密）一起调
if (espnow_get_key(key_info) != ESP_OK) esp_fill_random(key_info, APP_KEY_LEN);
espnow_set_key(key_info);
espnow_set_dec_key(key_info);
```

### 13. ESPNOW_INIT_CONFIG_DEFAULT 默认不开启各功能接收

```c
// ❌ WRONG — 以为 init 后 ota/control/debug 回调自动生效
espnow_init(&cfg); // receive_enable.* 大多为 0

// ✅ CORRECT — 需要的功能用对应模块 API 内部 set_config_for_data_type 开启，
//             或手动设 cfg.receive_enable.control_data = 1 等
espnow_config_t cfg = ESPNOW_INIT_CONFIG_DEFAULT();
cfg.receive_enable.control_bind = 1;
cfg.receive_enable.control_data = 1;
espnow_init(&cfg);
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 明确需求：是收发数据 / 控制 / OTA / 配网 / 调试 / 时间同步；确定 initiator 与 responder 角色 |
| 2 | Recipe | 在 `recipes/` 中匹配最接近的 recipe，遵循其调用链与初始化顺序 |
| 3 | Query | recipe 未覆盖的 API，查 `resources/api_reference.md`；配置项查 `resources/config_reference.md` |
| 4 | Validate | 校验函数签名、头文件包含（`espnow.h` / `espnow_ctrl.h` / `espnow_ota.h` …）、Kconfig 选项 |
| 5 | Confirm | 向用户呈现方案：依赖添加方式、Wi-Fi 模式、init 顺序、角色分配、回调注册 |
| 6 | Execute | 新工程：用 `idf.py create-project-from-example "espressif/esp-now=*:<example>"` 拉取最接近示例再改；已有工程：就地编辑 |
| 7 | Build | `idf.py set-target esp32c3` → `idf.py build` |
| 8 | Flash | `idf.py erase_flash` → `idf.py flash monitor` |
| 9 | Debug | 串口日志确认 init 顺序、`espnow_send` 返回值、事件回调是否触发；OTA/控制需两台设备联调 |

### Step 6 Detail — 工程创建策略

**新工程（目标目录无项目）：**

1. 按 README 用组件管理器加依赖：`idf.py add-dependency "espressif/esp-now=*"`。
2. 选最接近的官方示例作为起点：
   - 入门收发 → `examples/get-started`
   - 设备控制（开关+灯） → `examples/control`
   - 硬币电池开关 → `examples/coin_cell_demo/switch` + `examples/coin_cell_demo/bulb`
   - 批量 OTA → `examples/ota`
   - 安全加密 → `examples/security`
   - Wi-Fi 配网 → `examples/provisioning`
   - 无线调试 → `examples/wireless_debug`
   - 综合（控制+配网+时间同步） → `examples/solution`
3. 用 `idf.py create-project-from-example "espressif/esp-now=*:<name>"` 下载到当前目录，保留完整结构后修改。
4. 向用户说明复制了什么、为什么。

**已有工程：** 就地编辑，除非用户要求否则不覆盖。

---

## Failure Strategies

| Situation | Action |
|---|---|
| API 在 resources 中查不到 | 立即停止，告知用户该 API 不存在，勿臆造 |
| 不确定收不到数据 | 检查 Wi-Fi 是否 STA+start、是否 `espnow_set_config_for_data_type` 注册回调、`receive_enable` 是否开启 |
| 安全加密不生效 | 确认 `sec_enable=1` + 已 `espnow_set_key`/`set_dec_key` + `frame_head.security=true` |
| OTA 分发失败 | 确认 initiator 已把固件 `esp_ota_write` 到本地分区、SHA-256 正确、responder `espnow_ota_responder_start` 已调用 |
| 绑定无响应 | responder 是否进入 `espnow_ctrl_responder_bind` 窗口、RSSI 阈值是否过高、bindlist 是否已满（≤`ESPNOW_BIND_LIST_MAX_SIZE` 32） |
| 内存增长 | 检查所有 `*_result_free` / `*_scan_result_free` 是否成对调用；开启 `CONFIG_ESPNOW_MEM_DEBUG` 用 `espnow_mem_print_record` 排查 |
| 硬币电池发包复位 | 开启 `CONFIG_ESPNOW_LIGHT_SLEEP` + 发包前 light sleep 充电，检查 `esp_wifi_force_wakeup_acquire/release` 配对 |
| 不确定 IDF 版本 | 组件要求 `idf >= 4.4`（见 `idf_component.yml`）；v2.x.x 内置 IDF 从 v4.4 起 |

## References

- 场景 recipes → `recipes/` 目录
- API 速查 → `resources/api_reference.md`
- 配置参考 → `resources/config_reference.md`
- 常见陷阱 → `resources/pitfalls.md`
- 示例索引 → `resources/example_list.md`
- 仓库 User Guide → `User_Guide.md` / `User_Guide_CN.md`
- 仓库头文件 → `src/*/include/*.h`（API 权威来源）
