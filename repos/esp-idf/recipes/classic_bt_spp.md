# 经典蓝牙 SPP（串口透传）

> **适用摘要**: 用 Bluedroid 实现经典蓝牙 SPP（Serial Port Profile，基于 RFCOMM）：`esp_spp_enhanced_init` + `esp_spp_cfg_t` 选 CB/VFS 模式、`esp_spp_start_srv` 开服务、`esp_spp_write` 发数据、事件回调处理。**经典蓝牙需 `CONFIG_BT_CLASSIC_ENABLED=y` 且仅 esp32（双模）支持**，并需释放 BLE 控制器内存。适配自 `examples/bluetooth/bluedroid/classic_bt/bt_spp_acceptor`。

> Version: ESP-IDF version used by the project.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "经典蓝牙"
- "Classic BT"
- "SPP 串口"
- "蓝牙透传 / RFCOMM"
- "esp_spp"
- "蓝牙和手机配对"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | **仅 esp32**（双模 BT+BLE；esp32s2/c3/c6/h2 等无 Classic BT，不支持 SPP）。见 `SOC_BT_CLASSIC_SUPPORTED` |
| Kconfig | `CONFIG_BT_ENABLED=y`、`CONFIG_BT_BLUEDROID_ENABLED=y`、`CONFIG_BT_CLASSIC_ENABLED=y`。menuconfig：`Component config → Bluetooth → Bluedroid Options → Classic Bluetooth`；并启用 `BT SPP`（`CONFIG_BT_SPP_ENABLED=y`） |
| 组件 | `bt`、`nvs_flash` |
| 头文件 | `esp_bt.h`、`esp_bt_main.h`、`esp_gap_bt_api.h`（经典 GAP，注意非 `esp_gap_ble_api.h`）、`esp_bt_device.h`、`esp_spp_api.h` |
| 参考 | `examples/bluetooth/bluedroid/classic_bt/bt_spp_acceptor`（服务端/CB 模式）、`bt_spp_initiator`（客户端）、`bt_spp_vfs_acceptor`（VFS 模式） |

> **重要**：Classic BT 与 BLE 共用同一控制器，模式互斥。纯 Classic 应用需 `esp_bt_controller_mem_release(ESP_BT_MODE_BLE)` 释放 BLE 内存，并以 `ESP_BT_MODE_CLASSIC_BT` 启用控制器。

## 分步说明

### 服务端初始化（CB 模式：数据经回调送达）

SPP 有两种数据收发模式：`ESP_SPP_MODE_CB`（数据由回调推送，简单）与 `ESP_SPP_MODE_VFS`（注册为 VFS，用 `read/write` 文件描述符操作，吞吐高）。服务端常用 CB 模式起步。

```c
#include "esp_bt.h"
#include "esp_bt_main.h"
#include "esp_gap_bt_api.h"
#include "esp_bt_device.h"
#include "esp_spp_api.h"
#include "nvs_flash.h"
#include "esp_log.h"

#define SPP_TAG         "SPP_SRV"
#define SPP_SERVER_NAME "SPP_SERVER"

static const esp_spp_mode_t spp_mode = ESP_SPP_MODE_CB;
static const bool enable_l2cap_ertm = true;
static const esp_spp_sec_t sec_mask = ESP_SPP_SEC_AUTHENTICATE;
static const esp_spp_role_t role_slave = ESP_SPP_ROLE_SLAVE;
static uint32_t spp_conn_handle = 0;   /* 连接 handle，write 时用 */

static void esp_spp_cb(esp_spp_cb_event_t event, esp_spp_cb_param_t *param)
{
    switch (event) {
    case ESP_SPP_INIT_EVT:
        /* SPP 初始化完成，开服务（scn=0 让协议栈自动分配通道号） */
        if (param->init.status == ESP_SPP_SUCCESS) {
            esp_spp_start_srv(sec_mask, role_slave, 0, SPP_SERVER_NAME);
        }
        break;

    case ESP_SPP_START_EVT:
        /* 服务已启动，设设备名 + 可被发现可被连接 */
        if (param->start.status == ESP_SPP_SUCCESS) {
            esp_bt_gap_set_device_name(CONFIG_EXAMPLE_LOCAL_DEVICE_NAME);
            esp_bt_gap_set_scan_mode(ESP_BT_CONNECTABLE, ESP_BT_GENERAL_DISCOVERABLE);
            ESP_LOGI(SPP_TAG, "SPP server started, scn=%d", param->start.scn);
        }
        break;

    case ESP_SPP_SRV_OPEN_EVT:
        /* 远端连入 */
        spp_conn_handle = param->srv_open.handle;
        ESP_LOGI(SPP_TAG, "client connected, handle=%" PRIu32, spp_conn_handle);
        break;

    case ESP_SPP_DATA_IND_EVT:
        /* 收到数据（仅 CB 模式）。注意：回调里勿做耗时操作，否则堵塞协议栈 */
        ESP_LOGI(SPP_TAG, "recv %d bytes", param->data_ind.len);
        /* 回显 */
        esp_spp_write(param->data_ind.handle, param->data_ind.len, param->data_ind.data);
        break;

    case ESP_SPP_CONG_EVT:
        /* 拥塞状态变化（cong=true 时不要再 write，等 cong=false） */
        ESP_LOGI(SPP_TAG, "congestion status: %d", param->cong.cong);
        break;

    case ESP_SPP_WRITE_EVT:
        ESP_LOGI(SPP_TAG, "write complete, cong=%d", param->write.cong);
        break;

    case ESP_SPP_CLOSE_EVT:
        spp_conn_handle = 0;
        ESP_LOGI(SPP_TAG, "connection closed");
        break;

    default:
        break;
    }
}

/* 经典蓝牙 GAP 回调：配对/认证相关（PIN、SSP 数字比较等） */
static void esp_bt_gap_cb(esp_bt_gap_cb_event_t event, esp_bt_gap_cb_param_t *param)
{
    switch (event) {
    case ESP_BT_GAP_AUTH_CMPL_EVT:
        ESP_LOGI(SPP_TAG, "auth %s, device=%s",
                 param->auth_cmpl.stat == ESP_BT_STATUS_SUCCESS ? "success" : "failed",
                 param->auth_cmpl.device_name);
        break;
    case ESP_BT_GAP_PIN_REQ_EVT:
        /* 传统 PIN 配对：回 1234 */
        esp_bt_pin_code_t pin_code = {'1','2','3','4'};
        esp_bt_gap_pin_reply(param->pin_req.bda, true, 4, pin_code);
        break;
    case ESP_BT_GAP_CFM_REQ_EVT:
        /* SSP 数字比较：自动确认（演示用，生产环境应让用户确认） */
        esp_bt_gap_ssp_confirm_reply(param->cfm_req.bda, true);
        break;
    default:
        break;
    }
}

void app_main(void)
{
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    /* 仅用 Classic BT，释放 BLE 控制器内存 */
    ESP_ERROR_CHECK(esp_bt_controller_mem_release(ESP_BT_MODE_BLE));

    esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_bt_controller_init(&bt_cfg));
    ESP_ERROR_CHECK(esp_bt_controller_enable(ESP_BT_MODE_CLASSIC_BT));   /* 经典模式 */

    esp_bluedroid_config_t bluedroid_cfg = BT_BLUEDROID_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_bluedroid_init_with_cfg(&bluedroid_cfg));
    ESP_ERROR_CHECK(esp_bluedroid_enable());

    ESP_ERROR_CHECK(esp_bt_gap_register_callback(esp_bt_gap_cb));
    ESP_ERROR_CHECK(esp_spp_register_callback(esp_spp_cb));     /* SPP 单回调，所有事件走它 */

    /* v5.x 用 enhanced_init + esp_spp_cfg_t（取代旧的 esp_spp_init） */
    esp_spp_cfg_t bt_spp_cfg = {
        .mode = spp_mode,
        .enable_l2cap_ertm = enable_l2cap_ertm,
        .tx_buffer_size = 0,   /* 仅 VFS 模式有效 */
    };
    ESP_ERROR_CHECK(esp_spp_enhanced_init(&bt_spp_cfg));

    /* 配置 SSP（Secure Simple Pairing）IO 能力 */
    esp_bt_sp_param_t param_type = ESP_BT_SP_IOCAP_MODE;
    esp_bt_io_cap_t iocap = ESP_BT_IO_CAP_IO;
    esp_bt_gap_set_security_param(param_type, &iocap, sizeof(uint8_t));

    /* 配置 PIN 类型（VARIABLE 表示按需交互输入） */
    esp_bt_pin_type_t pin_type = ESP_BT_PIN_TYPE_VARIABLE;
    esp_bt_pin_code_t pin_code;
    esp_bt_gap_set_pin(pin_type, 0, pin_code);

    ESP_LOGI(SPP_TAG, "own BD addr: %s",
             bda2str((uint8_t *)esp_bt_dev_get_address(), ...));   /* 用 sprintf 格式化 6 字节地址 */
}
```

### 客户端要点（`bt_spp_initiator`）

客户端流程：`esp_spp_start_discovery(remote_bda)` 发现远端 SCN → `ESP_SPP_DISCOVERY_COMP_EVT` 拿到 SCN → `esp_spp_connect(sec_mask, ESP_SPP_ROLE_MASTER, remote_scn, peer_bda)` → `ESP_SPP_OPEN_EVT` 后 `esp_spp_write`。

```c
/* 客户端发起（节选） */
esp_spp_start_discovery(peer_bd_addr);   /* 异步，结果在 ESP_SPP_DISCOVERY_COMP_EVT */

/* 在 ESP_SPP_DISCOVERY_COMP_EVT 内 */
if (param->disc_comp.status == ESP_SPP_SUCCESS && param->disc_comp.scn_num > 0) {
    esp_spp_connect(ESP_SPP_SEC_AUTHENTICATE, ESP_SPP_ROLE_MASTER,
                    param->disc_comp.scn[0], peer_bd_addr);
}
```

### 关键 API

```c
/* esp_spp_api.h */
esp_err_t esp_spp_register_callback(esp_spp_cb_t cb);                       /* 单回调 */
esp_err_t esp_spp_enhanced_init(const esp_spp_cfg_t *cfg);                  /* v5.x 初始化 */
esp_err_t esp_spp_deinit(void);
esp_err_t esp_spp_start_srv(esp_spp_sec_t sec_mask, esp_spp_role_t role,
                            uint8_t local_scn, const char *name);           /* scn=0 自动分配 */
esp_err_t esp_spp_start_srv_with_cfg(const esp_spp_start_srv_cfg_t *cfg);   /* 带记录创建选项 */
esp_err_t esp_spp_stop_srv(void);
esp_err_t esp_spp_connect(esp_spp_sec_t sec_mask, esp_spp_role_t role,
                          uint8_t remote_scn, esp_bd_addr_t peer_bd_addr);
esp_err_t esp_spp_disconnect(uint32_t handle);
esp_err_t esp_spp_write(uint32_t handle, int len, uint8_t *p_data);         /* 仅 CB 模式 */
esp_err_t esp_spp_start_discovery(esp_bd_addr_t bd_addr);                   /* SDP 发现 SCN */
esp_err_t esp_spp_vfs_register(void);                                       /* VFS 模式注册 */
/* esp_gap_bt_api.h —— 经典蓝牙 GAP（注意非 BLE） */
esp_err_t esp_bt_gap_register_callback(esp_bt_gap_cb_t cb);
esp_err_t esp_bt_gap_set_device_name(const char *name);
esp_err_t esp_bt_gap_set_scan_mode(esp_bt_scan_mode_t mode);   /* ESP_BT_CONNECTABLE / general/limited */
```

关键 SPP 事件：`ESP_SPP_INIT_EVT`（初始化完成，开服务）→ `ESP_SPP_START_EVT`（服务就绪，设名+可发现）→ `ESP_SPP_SRV_OPEN_EVT`（客户端连入）→ `ESP_SPP_DATA_IND_EVT`（收到数据，仅 CB）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp_bt_controller_enable(CLASSIC_BT)` 报错或 SOC 不支持 | 芯片无 Classic BT | Classic BT 仅 esp32 支持；esp32c3/s3 等只能 BLE，改用 BLE SPP（`examples/bluetooth/bluedroid/ble/ble_spp_server`） |
| `CONFIG_BT_CLASSIC_ENABLED` 选不上 | 未选 Bluedroid Host | 经典蓝牙必须 `CONFIG_BT_BLUEDROID_ENABLED`（NimBLE 仅 BLE）；menuconfig 选 `Bluedroid - Dual-mode` |
| 编译报 `esp_spp_*` undefined | 未启用 SPP | `Component config → Bluetooth → Bluedroid Options → SPP`（`CONFIG_BT_SPP_ENABLED=y`） |
| 配对失败 | PIN/SSP 处理缺失 | 注册 GAP 回调处理 `ESP_BT_GAP_PIN_REQ_EVT` / `CFM_REQ_EVT`；设置 `set_pin` / `set_security_param` |
| 数据堵塞协议栈 | 在 `DATA_IND_EVT` 回调里做重活 | 回调内仅短处理；大数据搬移到应用任务，用队列转交 |
| `esp_spp_write` 返回忙 | 上一次写未完成（拥塞） | 收到 `ESP_SPP_WRITE_EVT` 且 `cong=false` 后再写；`cong=true` 时等 `ESP_SPP_CONG_EVT` 的 `cong=false` |
| 用了 BLE 的 GAP 头 | 协议栈不匹配 | Classic BT 用 `esp_gap_bt_api.h`（`esp_bt_gap_*`）；BLE 用 `esp_gap_ble_api.h`（`esp_ble_gap_*`） |

## 参考

- `examples/bluetooth/bluedroid/classic_bt/bt_spp_acceptor` — 服务端 + CB 模式（本 recipe 主来源）
- `examples/bluetooth/bluedroid/classic_bt/bt_spp_initiator` — 客户端（含消息解析）
- `examples/bluetooth/bluedroid/classic_bt/bt_spp_vfs_acceptor` — VFS 模式（高吞吐，`read/write` fd）
- ESP-IDF `components/bt/host/bluedroid/api/include/api/esp_spp_api.h`、`esp_gap_bt_api.h`、`components/bt/include/esp32/include/esp_bt.h`
- 文档 `docs/en/api-reference/bluetooth/esp_spp.rst`、`classic_bt.rst`、`esp_bt_device.rst`
