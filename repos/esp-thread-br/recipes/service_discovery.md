# 服务发现 (SRP + mDNS)

> **适用摘要**: Thread 设备经 SRP 注册服务，BR 通过 mDNS 转发到 Wi-Fi；反之 Wi-Fi mDNS 服务也能被 Thread 设备经 DNS 解析。

## 触发意图
- "服务发现"
- "SRP 注册服务"
- "mDNS avahi"
- "Thread 服务发布"

## 前置条件

| 条件 | 要求 |
|---|---|
| 设备 | ESP Thread BR + Thread CLI 设备 + Linux 主机 |
| 网络 | 主机与 BR 同 Wi-Fi；CLI 已加入 Thread 网络 |
| 参考 | `docs/en/codelab/service_discovery.rst`、`examples/basic_thread_border_router/README.md` |

## 分步说明

### 1. Thread → Wi-Fi：SRP 注册

CLI 设备上发布服务：
```
> ot srp client host name thread-device
> ot srp client host address auto
> ot srp client service add thread-service _test._tcp,_sub1,_sub2 12345 1 1 0778797a3d58595a
> ot srp client autostart enable
```

### 2. 在 BR 上验证 SRP Server 已收到

```
> ot srp server service
thread-service._test._tcp.default.service.arpa.
    deleted: false
    subtypes: _sub2,_sub1
    port: 12345
    priority: 1
    weight: 1
    ttl: 7200
    TXT: [xyz=58595a]
    host: thread-device.default.service.arpa.
    addresses: [fd66:afad:575f:1:744d:573e:6e60:188a]
```

### 3. Wi-Fi 侧 mDNS 浏览

BR 通过 mDNS 把服务桥接到 Wi-Fi。Linux 主机：
```bash
avahi-browse -rt _test._tcp
# hostname = [thread-device.local]
# address  = [fd66:afad:575f:1:744d:573e:6e60:188a]
# port     = [12345]
```

> BR 必须初始化 mDNS：`mdns_init()` + `mdns_hostname_set("esp-ot-br")`，`sdkconfig.defaults` 含 `CONFIG_MDNS_MAX_SERVICES=200`。

### 4. Wi-Fi → Thread：mDNS 发布 + DNS 解析

Linux 主机发布 mDNS 服务：
```bash
avahi-publish-service wifi-service _test._tcp 22222 test=1 dn="aabbbb"
# Established under name 'wifi-service'
```

取 BR 的 Mesh-Local Endpoint Identifier：
```
> ot ipaddr mleid
fdde:ad00:beef:0:f891:287:866:776
```

CLI 设备配 DNS 并解析：
```
> ot dns config fdde:ad00:beef:0:f891:287:866:776
> ot dns service wifi-service _test._tcp.default.service.arpa.
DNS service resolution response for wifi-service for service _test._tcp.default.service.arpa.
Port:22222, Priority:0, Weight:0, TTL:120
Host:FA001388.default.service.arpa.
HostAddress:fd33:1cc4:a6ec:2e0:2eea:7fff:fe37:b4fb TTL:120
TXT:[test=31, dn=616162626262] TTL:120
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| Wi-Fi 看不到 Thread 服务 | mDNS 未初始化或服务数上限太小 | 确认 `mdns_init()`；`CONFIG_MDNS_MAX_SERVICES` 调大 |
| `srp server service` 为空 | CLI 未 autostart | `ot srp client autostart enable` |
| DNS 解析失败 | CLI 未配 BR 的 mleid 作 DNS | `ot dns config <br-mleid>` |
| TXT 解析乱码 | 传的是十六进制 | `0778797a3d58595a` = `xyz=XYZ`，正常现象 |

## 参考
- `docs/en/codelab/service_discovery.rst`（3.3）
- `examples/basic_thread_border_router/main/esp_ot_br.c`（`mdns_init` / `mdns_hostname_set`）
- `examples/basic_thread_border_router/sdkconfig.defaults`（`CONFIG_MDNS_*`）
