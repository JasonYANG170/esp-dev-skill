# 组播转发 (Multicast Forwarding)

> **适用摘要**: 让 Wi-Fi/Ethernet 与 Thread 网络中处于同一组播组的设备相互可达（ICMP/UDP）。

## 触发意图
- "组播转发"
- "multicast forwarding"
- "ff04 组播"
- "跨 Thread Wi-Fi 组播"

## 前置条件

| 条件 | 要求 |
|---|---|
| 设备 | ESP Thread BR + Thread CLI 设备 + Linux 主机 |
| 网络 | 主机与 BR 同 Wi-Fi；CLI 已加入 Thread 网络 |
| 参考 | `docs/en/codelab/multicast_forwarding.rst` |

## 分步说明

### 1. 关键约束：组播组 scope

> 仅 admin-local (`ff04::/16` 及更大 scope) 及以上的组播会被 BR 转发；**link-local / realm-local 不会被转发**。

### 2. Thread CLI 加入组播组

```
> ot mcast join ff04::123
Done
```

> 该命令由 esp-thread-br 扩展 CLI `esp_ot_process_mcast_group`（`components/esp_ot_cli_extension`）提供，需 `CONFIG_OPENTHREAD_CLI_ESP_EXTENSION`。

### 3. 从 Wi-Fi 侧 ICMP 组播（Linux）

```bash
ping -I wlan0 -t 64 ff04::123
# 64 bytes from fd66:afad:575f:1:744d:573e:6e60:188a: icmp_seq=1 ttl=254 time=132 ms
```

### 4. 从 Wi-Fi 侧 UDP 组播

CLI 设备先开 UDP 监听：
```
> ot mcast join ff04::123
> ot udp open
> ot udp bind :: 5083
```

Linux 主机 `multicast_udp_client.py`：
```python
import socket
sock = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_BINDTODEVICE, b'wlan0')
sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_MULTICAST_HOPS, 32)
sock.sendto(b'hello', ('ff04::123', 5083))
```
```bash
python3 multicast_udp_client.py
```
CLI 设备将打印：`5 bytes from ... 56024 hello`

### 5. 从 Thread 侧发往 Wi-Fi 组播组

Linux 主机先开组播 UDP 服务端：
```python
import socket, struct
if_index = socket.if_nametoindex('wlan0')
sock = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
sock.bind(('::', 5083))
sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_JOIN_GROUP,
    struct.pack('16si', socket.inet_pton(socket.AF_INET6, 'ff04::123'), if_index))
while True:
    print(sock.recvfrom(1024))
```

CLI 发送：
```
> ot udp open
> ot udp send ff04::123 5083 hello
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 组播完全不通 | 用了 link-local (`ff02`) 或 realm-local | 改用 `ff04::` admin-local 及以上 |
| Wi-Fi 主机收不到 Thread 发来的组播 | 主机未 `IPV6_JOIN_GROUP` 加入该组 | Python 服务端先 join group |
| CLI 未识别 `mcast` 命令 | 未启用 ESP 扩展 CLI | 确认 `esp_cli_custom_command_init()` 已调用 |
| UDP bind 失败 | 端口被占 | 换端口 |

## 参考
- `docs/en/codelab/multicast_forwarding.rst`（3.2）
- `components/esp_ot_cli_extension/include/esp_ot_udp_socket.h`（`esp_ot_process_mcast_group`）
- `examples/basic_thread_border_router/README.md`（mDNS/SRP 依赖组播转发）
