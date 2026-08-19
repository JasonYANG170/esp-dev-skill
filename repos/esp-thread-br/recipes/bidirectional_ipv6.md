# 双向 IPv6 连通

> **适用摘要**: 让 Wi-Fi/Ethernet 主机与 Thread 设备互相用全局 IPv6 地址 ping 通；包含 Linux 主机 RA 接收配置。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-thread-br/resources/`, source/examples in `repos/esp-thread-br/`, and this recipe path `repos/esp-thread-br/recipes/bidirectional_ipv6.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图
- "Thread 设备 ping 不通"
- "双向 IPv6 连通"
- "accept_ra 配置"
- "Border Router 路由通告"

## 前置条件

| 条件 | 要求 |
|---|---|
| 设备 | ESP Thread BR + Thread CLI 设备 + Linux 主机 |
| 网络 | 主机与 BR 同一 Wi-Fi；Thread CLI 已加入 BR 形成的 Thread 网络 |
| 参考 | `docs/en/codelab/connectivity.rst`、`docs/en/codelab/basic_setup.rst` |

## 分步说明

### 1. 起网（BR 侧）

```
> wifi connect -s <ssid> -p <psk>     # 或 AUTO_START 自动连
> dataset init new
> dataset commit active
> ifconfig up
> thread start
> state
leader
```

### 2. CLI 设备加入网络

```
> ot dataset active -x      # 在 BR 上取 dataset
0e080000000000010000000300001835060004001fffe00208...
> ot dataset set active <hex>
> ot ifconfig up
> ot thread start
> ot state
router
```

### 3. 配置 Linux 主机接收 RA

```bash
sudo sysctl -w net/ipv6/conf/wlan0/accept_ra=2
sudo sysctl -w net/ipv6/conf/wlan0/accept_ra_rt_info_max_plen=128
```
> iOS 14+ / Android 8.1+ 手机会自动配置路由规则，无需手动。

### 4. 取 CLI 设备可路由地址并 ping

```
> ot ipaddr
fd66:afad:575f:1:744d:573e:6e60:188a    # 全局 OMR 地址（可路由）
fd87:8205:4651:27c8:0:ff:fe00:0
fe80:0:0:0:2433:db2e:62c:b2e4           # link-local，不可跨 BR
```

```bash
ping fd66:afad:575f:1:744d:573e:6e60:188a
# 64 bytes from ... icmp_seq=1 ttl=254 time=187 ms
```

### 5. 关键 lwIP 配置（确保 BR 发 RA + 转发）

BR 工程的 `sdkconfig.defaults` 必须含：
```bash
CONFIG_LWIP_IPV6_FORWARD=y
CONFIG_LWIP_HOOK_IP6_ROUTE_DEFAULT=y
CONFIG_LWIP_HOOK_ND6_GET_GW_DEFAULT=y
CONFIG_LWIP_HOOK_IP6_INPUT_CUSTOM=y
CONFIG_LWIP_IPV6_AUTOCONFIG=y
CONFIG_LWIP_HOOK_IP6_SELECT_SRC_ADDR_CUSTOM=y
CONFIG_LWIP_FORCE_ROUTER_FORWARDING=y
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| Linux 主机 ping 通但 `ot ping` 显示丢包 | OT 层 ping BR 自身 Wi-Fi 地址，回包被 lwIP 吞 | 设计行为（qa.rst 5.1），改在 lwIP 层 ping 或 ping 网内其他设备 |
| 主机收不到 RA | `accept_ra` 未设 2 | 设 `accept_ra=2` 且 `accept_ra_rt_info_max_plen=128` |
| CLI 设备无全局地址 | 未加入同一 Thread 网络 | 核对 dataset 一致、`state` 为 router/child |
| ping ttl=254 但高延迟 | 跨 BR 正常 | Thread 链路本身延迟较高，正常 |

## 参考
- `docs/en/codelab/connectivity.rst`（3.1 Bi-directional IPv6）
- `docs/en/codelab/basic_setup.rst`（3.0 网络拓扑）
- `docs/en/qa.rst`（5.1 ping 看似丢包）
- `examples/basic_thread_border_router/sdkconfig.defaults`
