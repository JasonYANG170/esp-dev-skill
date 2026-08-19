# 自定义 Cluster（custom cluster）

> **适用摘要**: 讲解如何创建厂商/私有 cluster——注册自定义命令处理回调（`ezb_zcl_custom_cluster_handlers_t`）、定义私有 cluster ID / 命令 ID / 属性、发送与接收自定义命令。基于 `examples/customized_devices/` 的 data stream（producer/consumer）模式。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "自定义 cluster"
- "厂商私有命令"
- "custom cluster 命令收发"
- "data_producer / data_consumer"
- "扩展 Zigbee 设备"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `#include "esp_zigbee.h"`（含 `ezbee/zcl/cluster/custom.h`、`ezbee/zcl.h`） |
| 参考项目 | `examples/customized_devices/data_producer/`、`data_consumer/` |

## 分步说明

### 1. 定义私有 cluster / 命令 / 属性 ID

```c
// 示例：data stream cluster（参见 examples/customized_devices）
#define ESP_ZIGBEE_CUSTOMIZED_DATA_STREAM_CLUSTER_ID          0xFFF0   // 厂商自定义 cluster 区段
#define ESP_ZIGBEE_CUSTOMIZED_DATA_STREAM_QUERY_DATA_REQ_CMD_ID  0x00
#define ESP_ZIGBEE_CUSTOMIZED_DATA_STREAM_QUERY_DATA_RSP_CMD_ID  0x01
#define ESP_ZIGBEE_CUSTOMIZED_DATA_STREAM_DATA_LENGTH_ATTR_ID    0x0000
#define ESP_ZIGBEE_CUSTOMIZED_DATA_STREAM_DATA_BEGIN_ATTR_ID     0x0001
```

> Zigbee 厂商自定义 cluster ID 通常落在 `0xFC00–0xFFFF`。

### 2. 注册命令处理 handlers

`ezb_zcl_custom_cluster_handlers_t` 含 `process_cmd_cb`、`check_value_cb`、`write_attr_cb`、`cmd_disc_cb`。`process_cmd_cb` 接收 `ezb_zcl_cmd_hdr_t` + payload。

```c
static ezb_zcl_status_t customized_cmd_handler(const ezb_zcl_cmd_hdr_t *header,
                                               const uint8_t *payload, uint16_t payload_length)
{
    ezb_zcl_status_t ret = EZB_ZCL_STATUS_SUCCESS;
    if (header->cluster_id != ESP_ZIGBEE_CUSTOMIZED_DATA_STREAM_CLUSTER_ID) {
        return EZB_ZCL_STATUS_UNSUPPORTED_CLUSTER;
    }
    if (EZB_ZCL_CMD_FC_IS_TO_CLI_DIRECTION(header->fc)) {
        return EZB_ZCL_STATUS_INVALID_FIELD;
    }
    switch (header->cmd_id) {
    case ESP_ZIGBEE_CUSTOMIZED_DATA_STREAM_QUERY_DATA_REQ_CMD_ID:
        ret = (send_query_data_response(header) == EZB_ERR_NONE)
                  ? EZB_ZCL_STATUS_SUCCESS : EZB_ZCL_STATUS_UNSUP_CMD;
        break;
    default:
        ret = EZB_ZCL_STATUS_UNSUP_CMD;
        break;
    }
    /* 失败时主动回 default response */
    if (ret != EZB_ZCL_STATUS_SUCCESS) {
        zcl_default_rsp_cmd_req(header, ret);
    }
    return ret;
}

static uint8_t customized_disc_cmd_handler(bool is_recv, uint8_t **list)
{
    static uint8_t recv_cmd_list[] = { ESP_ZIGBEE_CUSTOMIZED_DATA_STREAM_QUERY_DATA_REQ_CMD_ID };
    static uint8_t send_cmd_list[] = { ESP_ZIGBEE_CUSTOMIZED_DATA_STREAM_QUERY_DATA_RSP_CMD_ID };
    *list = is_recv ? recv_cmd_list : send_cmd_list;
    return is_recv ? sizeof(recv_cmd_list) : sizeof(send_cmd_list);
}

void esp_zigbee_zcl_custom_cluster_init(uint8_t ep_id)
{
    ezb_zcl_custom_cluster_handlers_t handlers = {
        .cluster_id     = ESP_ZIGBEE_CUSTOMIZED_DATA_STREAM_CLUSTER_ID,
        .cluster_role   = EZB_ZCL_CLUSTER_SERVER,
        .process_cmd_cb = customized_cmd_handler,
        .check_value_cb = NULL,
        .write_attr_cb  = NULL,
        .cmd_disc_cb    = customized_disc_cmd_handler,
    };
    ezb_zcl_custom_cluster_handlers_register(&handlers);   // 注册到协议栈（custom.h）
}
```

> handlers 结构体字段（`ezbee/zcl/cluster/custom.h`）：`cluster_id`、`cluster_role`、`process_cmd_cb`、`check_value_cb`、`write_attr_cb`、`cmd_disc_cb`。注册函数为 `ezb_zcl_custom_cluster_handlers_register`。

### 3. 构造响应/发起命令（`ezb_zcl_custom_cluster_cmd_req`）

`cmd_ctrl` 用 `CMD_HDR_TO_RSP(header)` 反向构造响应。

```c
static ezb_err_t send_query_data_response(const ezb_zcl_cmd_hdr_t *header)
{
    ezb_zcl_custom_cluster_cmd_t cmd_req = {
        .cmd_ctrl    = CMD_HDR_TO_RSP(header),            // src/dst ep、addr、cluster_id 取反
        .cmd_id      = ESP_ZIGBEE_CUSTOMIZED_DATA_STREAM_QUERY_DATA_RSP_CMD_ID,
        .data_length = 1024,
        .data        = (void *)generate_data(header->dst_ep),
    };
    return ezb_zcl_custom_cluster_cmd_req(&cmd_req);
}
```

### 4. 读写自定义属性（`ezb_zcl_get_attr_desc` + `ezb_zcl_attr_desc_get_value`）

```c
ezb_zcl_attr_desc_t *desc =
    ezb_zcl_get_attr_desc(ep_id, ESP_ZIGBEE_CUSTOMIZED_DATA_STREAM_CLUSTER_ID,
                          EZB_ZCL_CLUSTER_SERVER,
                          ESP_ZIGBEE_CUSTOMIZED_DATA_STREAM_DATA_LENGTH_ATTR_ID,
                          EZB_ZCL_STD_MANUF_CODE);
assert(desc);
uint16_t value = 0;
ezb_zcl_attr_desc_get_value(desc, &value);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 自定义命令无响应 | 未注册 handlers / cluster_id 不匹配 | 调注册函数；handler 内校验 cluster_id |
| 响应方向错 | `CMD_HDR_TO_RSP` 方向反 | 响应方向 = 请求取反 |
| 远端识别为未知 cluster | cluster_id 不在自定义区段 | 用 `0xFC00–0xFFFF` |
| 属性读不到 | attr 未加到 cluster desc | 确认 desc 创建并 add 到 endpoint |
| `EZB_ZCL_STATUS_INVALID_FIELD` | 响应方向判错 | 用 `EZB_ZCL_CMD_FC_IS_TO_CLI_DIRECTION` 判断 |

## 参考项目

- `examples/customized_devices/data_producer/main/data_stream_server.c` — server 端命令处理
- `examples/customized_devices/data_consumer/main/data_stream_client.c` — client 端发起命令
- `examples/customized_devices/data_producer/main/data_producer.c` — 设备创建与 cluster 注册
- `components/esp-zigbee-lib/include/ezbee/zcl/cluster/custom.h`
