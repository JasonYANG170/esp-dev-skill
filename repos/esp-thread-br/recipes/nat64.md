# NAT64 / DNS64 访问 IPv4 互联网

> **适用摘要**: 让 Thread 设备经 BR 的 NAT64 访问 IPv4 互联网（如 `curl http://www.espressif.com`）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-thread-br/resources/`, source/examples in `repos/esp-thread-br/`, and this recipe path `repos/esp-thread-br/recipes/nat64.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图
- "NAT64"
- "DNS64"
- "Thread 设备访问 IPv4"
- "ot curl"

## 前置条件

| 条件 | 要求 |
|---|---|
| 设备 | ESP Thread BR + Thread CLI 设备 |
| 网络 | CLI 已加入 Thread 网络；BR backbone 联网 |
| 扩展 CLI | `CONFIG_OPENTHREAD_DNS64_CLIENT=y`（`dns64server`）、`curl` 命令 |
| 参考 | `docs/en/codelab/nat64.rst` |

## 分步说明

### 1. 配置 DNS64 服务器（CLI 设备）

`dns64server` 命令由 `esp_openthread_process_dns64_server`（`components/esp_ot_cli_extension/include/esp_ot_dns64.h`）提供，需 `CONFIG_OPENTHREAD_DNS64_CLIENT=y`：

```
> ot dns64server 8.8.8.8
Done
```

### 2. 通过 NAT64 curl 访问 IPv4 网站

`curl` 命令由 `esp_openthread_process_curl`（`esp_ot_curl.h`）提供：

```
> ot curl http://www.espressif.com
Done
I (22289) HTTP_CLIENT: Body received in fetch header state, ...
<html>
<head><title>301 Moved Permanently</title></head>
...
```

> 流程：Thread 设备先向 IPv4 DNS（经 BR NAT64 翻译）解析域名，再经 BR NAT64 取页面。

### 3. NAT64 由 Border Router 库自动启用

成功起网后日志可见：
```
I (12401) OPENTHREAD: NAT64 ready
```
NAT64 由 `esp_openthread_border_router_init()` 初始化的 Border Router 库提供，无需额外 API 调用。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `dns64server` 命令不存在 | 未启用 DNS64 client | `CONFIG_OPENTHREAD_DNS64_CLIENT=y` |
| curl 超时 | backbone 未联网 / DNS 不可达 | 确认 BR Wi-Fi 能上网，DNS IP 可达 |
| `NAT64 ready` 未出现 | BR 未正确 init | 核对 `esp_openthread_border_router_init()` 已调用、backbone 已绑定 |
| 看不到 body | 目标站点 301/302 | 正常，curl 已收到响应 |

## 参考
- `docs/en/codelab/nat64.rst`（3.4）
- `components/esp_ot_cli_extension/include/esp_ot_dns64.h`
- `components/esp_ot_cli_extension/include/esp_ot_curl.h`
