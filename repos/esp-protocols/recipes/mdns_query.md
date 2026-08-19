# mDNS 服务查询（Query）

> **适用摘要**: 用 mDNS 查询局域网服务与主机，包括 PTR（服务）、SRV、TXT、A/AAAA 记录，遍历结果链表并释放。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-protocols/resources/`, source/examples in `repos/esp-protocols/`, and this recipe path `repos/esp-protocols/recipes/mdns_query.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "mDNS 查询服务"
- "mdns query"
- "查找局域网 _http._tcp 设备"
- "mdns_query_ptr / mdns_query_a"
- "解析 .local 主机名"

## 前置条件

| 条件 | 要求 |
|---|---|
| 网络 | 已连 Wi-Fi 或以太网 |
| 组件依赖 | `espressif/mdns` |
| 参考示例 | `components/mdns/examples/query_advertise/main/mdns_example_main.c` |

## 分步说明

### 1. 查询服务（PTR → SRV/TXT/A/AAAA）

```c
#include "mdns.h"
#include "esp_log.h"

static const char *TAG = "mdns";

static void mdns_print_results(mdns_result_t *results)
{
    mdns_result_t *r = results;
    while (r) {
        if (r->instance_name)
            printf("  PTR : %s.%s.%s\n", r->instance_name, r->service_type, r->proto);
        if (r->hostname)
            printf("  SRV : %s.local:%u\n", r->hostname, r->port);
        for (int t = 0; t < r->txt_count; t++)
            printf("  TXT : %s=%s\n", r->txt[t].key, r->txt[t].value ? r->txt[t].value : "NULL");
        for (mdns_ip_addr_t *a = r->addr; a; a = a->next) {
            if (a->addr.type == ESP_IPADDR_TYPE_V6)
                printf("  AAAA: " IPV6STR "\n", IPV62STR(a->addr.u_addr.ip6));
            else
                printf("  A   : " IPSTR "\n", IP2STR(&a->addr.u_addr.ip4));
        }
        r = r->next;
    }
}

static void query_mdns_service(const char *service_name, const char *proto)
{
    mdns_result_t *results = NULL;
    esp_err_t err = mdns_query_ptr(service_name, proto, 3000, 20, &results);
    if (err) {
        ESP_LOGE(TAG, "Query Failed: %s", esp_err_to_name(err));
        return;
    }
    if (!results) {
        ESP_LOGW(TAG, "No results");
        return;
    }
    mdns_print_results(results);
    mdns_query_results_free(results);   // 必须释放
}
```

### 2. 解析主机名为 IPv4 / IPv6

```c
    esp_ip4_addr_t addr4;
    if (mdns_query_a("esp32-xxxx", 3000, &addr4) == ESP_OK) {
        ESP_LOGI(TAG, "A: " IPSTR, IP2STR(&addr4));
    }

    esp_ip6_addr_t addr6;
    if (mdns_query_aaaa("esp32-xxxx", 3000, &addr6) == ESP_OK) {
        ESP_LOGI(TAG, "AAAA: " IPV6STR, IPV62STR(addr6));
    }
```

### 3. 单独查询 SRV / TXT

```c
    mdns_result_t *res = NULL;
    if (mdns_query_srv("ESP32-WebServer", "_http", "_tcp", 3000, &res) == ESP_OK) {
        ESP_LOGI(TAG, "SRV hostname=%s port=%u", res->hostname, res->port);
        mdns_query_results_free(res);
    }

    if (mdns_query_txt("ESP32-WebServer", "_http", "_tcp", 3000, &res) == ESP_OK) {
        for (int t = 0; t < res->txt_count; t++)
            ESP_LOGI(TAG, "TXT %s=%s", res->txt[t].key, res->txt[t].value);
        mdns_query_results_free(res);
    }
```

### 4. 通用查询（`mdns_query`）

```c
    mdns_result_t *results = NULL;
    // name=NULL 表示按 service_type 查 PTR；type=MDNS_TYPE_PTR
    mdns_query(NULL, "_http", "_tcp", MDNS_TYPE_PTR, 3000, 20, &results);
    mdns_query_results_free(results);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 内存持续增长 | 查询结果链表未释放 | 每次查询后 `mdns_query_results_free()` |
| `mdns_query_ptr` 超时无结果 | 服务端未发布 / 不在同域 | 对端先 `mdns_service_add`；同网段放通 UDP 5353 |
| `mdns_query_a` 返回 ESP_ERR_NOT_FOUND | hostname 不存在或还没通告 | 等待几秒重试（TTL/通告周期） |
| 结果里 `addr` 为空 | 该实例尚未注册 A/AAAA | 服务端确认 IP 已下发到 mDNS |
| max_results 太小 | 漏掉部分设备 | 增大 `max_results` 参数 |

## 参考

- `components/mdns/examples/query_advertise/main/mdns_example_main.c` — `mdns_print_results` / `query_mdns_service` 实现
- `components/mdns/include/mdns.h` — `mdns_query_ptr/srv/txt/a/aaaa` / `mdns_query` / `mdns_query_results_free`
- `docs/mdns/en/index.rst` — mDNS 文档入口
