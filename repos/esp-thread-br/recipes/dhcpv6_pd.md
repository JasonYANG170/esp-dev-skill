# DHCPv6 Prefix Delegation (PD) 客户端

> **适用摘要**: 让 BR 作为 DHCPv6 PD 客户端，从网络中的 DHCPv6 服务器（如 Kea）申请一段 IPv6 前缀，并下发给 Thread 设备，使其获得可全局路由的 IPv6 地址。

## 触发意图
- "DHCPv6 Prefix Delegation"
- "DHCPv6 PD"
- "ot br pd"
- "omrprefix / on-link prefix 下发"
- "Thread 设备获取全局 IPv6 前缀"

## 前置条件

| 条件 | 要求 |
|---|---|
| 设备 | ESP Thread BR + Thread CLI 设备 + Linux 主机（运行 DHCPv6 服务器） |
| 网络 | BR 与 Linux 主机接入同一 Wi-Fi/Ethernet；Thread CLI 设备已加入 BR 形成的 Thread 网络 |
| BR 状态 | 已为 `leader`，backbone 联网（见 `recipes/build_and_run.md`） |
| 参考 | `docs/en/codelab/dhcpv6_pd.rst`（3.8）、`examples/basic_thread_border_router/` |

> 说明：DHCPv6 PD 是 Thread 1.4 边界路由器特性，由 OpenThread Border Routing 模块实现，运行时通过 BR 上的 `ot br pd` CLI 控制，无需额外 Kconfig。

## 分步说明

### 1. 在 Linux 主机安装并配置 Kea DHCPv6 服务器

Kea 是 ISC 提供的开源 DHCPv6 服务器，本步骤将其配置为向 BR 下发 `/64` 前缀（从一个 `/48` 池中委派）。

```bash
sudo apt-get update
sudo apt-get install kea-dhcp6-server
```

编辑 `/etc/kea/kea-dhcp6.conf`（把 `wlan0` 换成与 BR 同网段的网卡名）：

```json
{
    "Dhcp6": {
        "interfaces-config": {
            "interfaces": ["wlan0"]
        },
        "lease-database": {
            "type": "memfile",
            "persist": true,
            "name": "/var/lib/kea/dhcp6.leases"
        },
        "preferred-lifetime": 3000,
        "valid-lifetime": 4000,
        "subnet6": [
            {
                "subnet": "2001:db8::/64",
                "pools": [],
                "pd-pools": [
                    {
                        "prefix": "2001:db8:1::",
                        "prefix-len": 48,
                        "delegated-len": 64
                    }
                ]
            }
        ]
    }
}
```

启动并设为开机自启：

```bash
sudo systemctl start kea-dhcp6-server
sudo systemctl enable kea-dhcp6-server
sudo systemctl status kea-dhcp6-server      # 确认 active (running)
```

> `pd-pools` 字段：`prefix` + `prefix-len` 定义可委派的大前缀池（这里是 `2001:db8:1::/48`），`delegated-len` 指每次实际下发给客户端的前缀长度（这里 `/64`）。

### 2. 在 BR 上启用 DHCPv6 PD 客户端

BR 正常起网（`leader`）后，通过串口 CLI 启用 PD 客户端。`ot br pd` 是标准 OpenThread Border Router CLI（`ot cli`），运行时即生效：

```
> ot br pd enable
Done
```

### 3. 验证 BR 已收到并下发委派前缀

```
> ot br pd state
running
Done

> ot br pd omrprefix
2001:db8:1:3::/64 lifetime:3120 preferred:3000
Done
```

- `state` 返回 `running` 表示 PD 客户端正在运行且已拿到前缀。
- `omrprefix` 输出 On-Mesh Route Prefix（即下发给 Thread 网络的前缀）及其剩余 lifetime / preferred lifetime（单位秒，随租约递减）。

> 对应的 CLI 还有 `ot br pd onlinkprefix`（查询 BR 通告的 on-link prefix）、`ot br pd disable`。这些命令由 OpenThread Border Routing 的 DHCPv6 PD 子模块实现，见 https://openthread.io/reference。

### 4. 验证 Thread 设备获得委派前缀内的地址

在 Thread CLI 设备上查看 IPv6 地址，应能看到落在 `omrprefix` 范围内的全局地址：

```
esp32c6> ot ipaddr
2001:db8:1:3:eae1:e8ce:1f85:d1dd     ← 来自委派前缀 2001:db8:1:3::/64
fd5b:281f:1ef6:923c:0:ff:fe00:d001   ← RLOC
fd5b:281f:1ef6:923c:37b4:c051:fe3d:41b3  ← mesh-local EID
fe80:0:0:0:5059:8382:b52:5f0         ← link-local
Done
```

### 5. 从 Linux 主机 ping 委派前缀地址

```bash
ping 2001:db8:1:3:eae1:e8ce:1f85:d1dd
PING 2001:db8:1:3:eae1:e8ce:1f85:d1dd(2001:db8:1:3:eae1:e8ce:1f85:d1dd) 56 data bytes
64 bytes from 2001:db8:1:3:eae1:e8ce:1f85:d1dd: icmp_seq=1 ttl=254 time=1734 ms
64 bytes from 2001:db8:1:3:eae1:e8ce:1f85:d1dd: icmp_seq=2 ttl=254 time=682 ms
...
```

收到回包即证明：前缀已下发 → Thread 设备已配置全局地址 → 经 BR 路由可达骨干网。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ot br pd state` 显示非 `running` | Kea 未运行或未配 `pd-pools` | `systemctl status kea-dhcp6-server`；确认 `pd-pools` 的 `delegated-len` ≤ `prefix-len` 的可用位 |
| BR 拿不到前缀 | Kea 监听网卡与 BR 不在同网段 | `interfaces` 改成与 BR 同 Wi-Fi/Ethernet 的接口；关闭该网卡上的 IPv6 privacy/RA block |
| `ot br pd omrprefix` 为空 | PD 已启用但租约未到 | 等几秒重新查询；确认 BR 已为 `leader` 且 backbone 联网 |
| Thread 设备无委派前缀地址 | 设备尚未刷新 RA / SLAAC | `ot ipaddr` 多查几次；确认 `omrprefix` 已返回前缀 |
| Linux 主机 ping 不通 | 主机未接受 RA 路由信息 | Linux：`sysctl net.ipv6.conf.<if>.accept_ra_rt_info_max_plen=128` |
| `ot br pd enable` 报错或无此命令 | OpenThread 编译未含 DHCPv6 PD | 默认 BR 配置已含；自定义构建时确认 `CONFIG_OPENTHREAD_BORDER_ROUTER=y` 且 OpenThread 版本支持 PD |

## 参考项目
- `docs/en/codelab/dhcpv6_pd.rst`（3.8）— Kea 配置与完整验证流程
- `docs/en/codelab/index.rst` — Codelab 目录（含 `dhcpv6_pd` 条目）
- `examples/basic_thread_border_router/` — 默认支持 DHCPv6 PD（运行时 CLI 控制）
- `docs/en/index.rst` — 特性列表含 DHCPv6 PD
