# mDNS 服务发布（Advertise）

> **适用摘要**: 初始化 mDNS，设置 hostname 与实例名，用 `mdns_service_add()` 发布服务（如 `_http._tcp`），配置 TXT 记录、子类型（subtype）与委托主机（delegated host）。

## 触发意图

- "mDNS 发布服务"
- "mdns advertise"
- "_http._tcp 服务"
- "局域网服务发现"
- "mdns_service_add"

## 前置条件

| 条件 | 要求 |
|---|---|
| 网络 | 已连 Wi-Fi 或以太网（mDNS 跑在 IP 之上） |
| 组件依赖 | `espressif/mdns`（>=1.11.2） |
| 参考示例 | `components/mdns/examples/query_advertise/main/mdns_example_main.c` |

## 分步说明

### 1. 初始化 mDNS、设置 hostname 与实例名

```c
#include "mdns.h"
#include "esp_log.h"

static const char *TAG = "mdns";

static void initialise_mdns(void)
{
    ESP_ERROR_CHECK(mdns_init());                       // 必须先初始化
    ESP_ERROR_CHECK(mdns_hostname_set("esp32-xxxx"));   // 设置 hostname（广告服务前提）
    ESP_ERROR_CHECK(mdns_instance_name_set("ESP32-WebServer"));
}
```

> hostname 必须在 `mdns_service_add()` 之前设置，否则发布失败。

### 2. 发布带 TXT 记录的服务

```c
    mdns_txt_item_t serviceTxtData[3] = {
        {"board", "esp32"},
        {"u", "user"},
        {"p", "password"}
    };

    // instance_name, service_type, proto, port, txt[], num_items
    ESP_ERROR_CHECK(mdns_service_add("ESP32-WebServer", "_http", "_tcp", 80, serviceTxtData, 3));

    // 添加 service subtype（如 _server）
    ESP_ERROR_CHECK(mdns_service_subtype_add_for_host("ESP32-WebServer", "_http", "_tcp", NULL, "_server"));
```

### 3. 动态修改 TXT 项

```c
    // 新增一个 TXT 项
    ESP_ERROR_CHECK(mdns_service_txt_item_set("_http", "_tcp", "path", "/foobar"));

    // 修改已有 TXT 项的值（显式长度，可含不可打印字符）
    ESP_ERROR_CHECK(mdns_service_txt_item_set_with_explicit_value_len(
        "_http", "_tcp", "u", "admin", strlen("admin")));
```

### 4. 委托主机（Delegated Host）发布

为其他主机代理发布服务（A/AAAA 查询会回复该主机的地址）：

```c
    mdns_ip_addr_t addr4, addr6;
    esp_netif_str_to_ip4("10.0.0.1", &addr4.addr.u_addr.ip4);
    addr4.addr.type = ESP_IPADDR_TYPE_V4;
    esp_netif_str_to_ip6("fd11:22::1", &addr6.addr.u_addr.ip6);
    addr6.addr.type = ESP_IPADDR_TYPE_V6;
    addr4.next = &addr6;
    addr6.next = NULL;

    ESP_ERROR_CHECK(mdns_delegate_hostname_add("myhost-delegated", &addr4));
    ESP_ERROR_CHECK(mdns_service_add_for_host(
        "test0", "_http", "_tcp", "myhost-delegated", 1234, serviceTxtData, 3));
```

### 5. 其它管理 API

```c
mdns_service_port_set("_http", "_tcp", 8080);              // 改端口
mdns_service_txt_item_remove("_http", "_tcp", "p");        // 删 TXT 项
mdns_service_remove("_http", "_tcp");                      // 移除服务
mdns_service_remove_all();                                 // 移除全部服务
mdns_free();                                               // 停止并释放 mDNS
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `mdns_service_add` 返回 `ESP_FAIL` | 未先 `mdns_init()` 或未设 hostname | 先 init + hostname_set |
| 服务发布后别的设备查不到 | 不在同一多播域 / 防火墙挡了 5353 | 确认同网段；放通 UDP 5353 |
| TXT 记录不更新 | 用了错误的 service_type/proto | service_type/proto 必须与 add 时完全一致 |
| 多实例同类型失败 | 未启用 `CONFIG_MDNS_MULTIPLE_INSTANCE` | menuconfig 开启后才能 add 多个同类型实例 |
| RAM 占用高 | 服务/接口数过多 | 调小 `CONFIG_MDNS_MAX_SERVICES` / `CONFIG_MDNS_MAX_INTERFACES` |

## 参考

- `components/mdns/examples/query_advertise/main/mdns_example_main.c` — 发布 + 查询完整示例
- `components/mdns/include/mdns.h` — `mdns_service_add` / TXT / subtype / delegate API
- `components/mdns/Kconfig` — `MDNS_MAX_SERVICES` / `MDNS_MAX_INTERFACES` / `MDNS_MULTIPLE_INSTANCE`
