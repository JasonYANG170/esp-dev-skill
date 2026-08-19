# 单播（peer）与分组控制

> **适用摘要**: 使用 `espnow_add_peer` 实现单播定向发送，使用 `espnow_add_group` / `espnow_set_group` 动态管理设备分组与组播（基于 `espnow.h` 真实 API）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-now/resources/`, source/examples in `repos/esp-now/`, and this recipe path `repos/esp-now/recipes/unicast_and_group.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "ESP-NOW 单播"
- "指定 MAC 发送"
- "分组控制"
- "组播 group"
- "添加 peer"

## 前置条件

| 条件 | 要求 |
|---|---|
| 初始化 | 已 `espnow_init(&cfg)` |
| 参考头文件 | `src/espnow/include/espnow.h` |

## 分步说明

### 1. 单播：添加 peer 并定向发送

```c
#include "espnow.h"

espnow_addr_t peer_addr = {0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF};
uint8_t lmk[16] = {0};   // 本地主密钥，可 NULL 或 16 字节

// 添加 peer（用于 unicast 路由）
esp_err_t ret = espnow_add_peer(peer_addr, lmk);  // lmk 可传 NULL
ESP_ERROR_CHECK(ret);

// 定向发送：dest_addr 传 peer_addr，frame_head.broadcast = false
espnow_frame_head_t fh = {
    .broadcast        = false,   // 单播
    .ack              = true,    // 等待 ACK 保证可靠
    .retransmit_count = 10,
};
uint8_t msg[] = "hello unicast";
espnow_send(ESPNOW_DATA_TYPE_DATA, peer_addr, msg, sizeof(msg), &fh, portMAX_DELAY);

// 删除 peer
espnow_del_peer(peer_addr);
```

> `lmk` 为 peer 本地主密钥，用于底层 Wi-Fi 加密，可传 `NULL` 或 `ESP_NOW_KEY_LEN` 长度数据（见头文件 `espnow_add_peer` 注释）。

### 2. 本地分组管理

```c
espnow_group_t group_id = {0x01, 0x02, 0x03, 0x04, 0x05, 0x06};

// 添加本机到某分组
espnow_add_group(group_id);

// 查询本机是否属于某分组
if (espnow_is_my_group(group_id)) {
    ESP_LOGI(TAG, "I am in this group");
}

// 获取分组数量与列表
int num = espnow_get_group_num();
espnow_group_t list[8];
espnow_get_group_list(list, num);

// 删除分组
espnow_del_group(group_id);
```

### 3. 动态分组（通过命令下发，改变其它设备的分组）

```c
espnow_addr_t addrs_list[2] = {
    {0xAA,0xBB,0xCC,0xDD,0xEE,0x01},
    {0xAA,0xBB,0xCC,0xDD,0xEE,0x02},
};
espnow_group_t group_id = {0x10,0x20,0x30,0x40,0x50,0x60};
espnow_frame_head_t fh = ESPNOW_FRAME_CONFIG_DEFAULT();

// enable=true 添加分组，false 删除分组；wait_ticks 控制发送超时
espnow_set_group(addrs_list, 2, group_id, &fh, true, portMAX_DELAY);
```

### 4. 组播发送（broadcast=true + group=true）

```c
espnow_frame_head_t fh = {
    .broadcast        = true,
    .group            = true,    // 仅组广播模式有效
    .retransmit_count = 5,
};
espnow_send(ESPNOW_DATA_TYPE_GROUP, group_id, payload, size, &fh, portMAX_DELAY);
```

> 帧头字段含义见 `espnow_frame_head_t`：`broadcast`/`group`/`ack`/`retransmit_count`(≤0x1f)/`forward_ttl`(≤0x1f)/`forward_rssi`/`security`/`channel`(`ESPNOW_CHANNEL_CURRENT=0` 或 `ESPNOW_CHANNEL_ALL=0x0f`)/`filter_weak_signal`/`filter_adjacent_channel`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 单播发不出 | 未 `espnow_add_peer` | 先添加 peer 再发送 |
| peer 删除后仍能收 | 对端缓存 | 重启或双方都 `espnow_del_peer` |
| 组播收不到 | 本机未 `espnow_add_group` 或 `receive_enable.group=0` | `cfg.receive_enable.group=1` 且本机加入该组 |
| `forward_ttl`/`retransmit_count` 超限 | 超过 0x1f(31) | 用 `ESPNOW_RETRANSMIT_MAX_COUNT` / `ESPNOW_FORWARD_MAX_COUNT` 上限 |
| 邻道干扰 | 未开 `filter_adjacent_channel` | 设 `frame_head.filter_adjacent_channel=1` |

## 参考

- `src/espnow/include/espnow.h` — `espnow_add_peer` / `espnow_del_peer` / `espnow_add_group` / `espnow_set_group` / `espnow_frame_head_t`
- `examples/solution/` — 综合示例含分组与多设备控制
