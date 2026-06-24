# Shell sta 命令：连接 AP

> **适用摘要**: 用 `WASMachine> sta` 命令让设备以 Station 模式连接 AP，为远程 App 管理（`host_tool`）与联网 native（HTTP/MQTT/RainMaker）准备网络。

## 触发意图

- "连 Wi-Fi"
- "sta 命令"
- "host_tool 连不上设备"
- "给 wasmachine 配网"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_WASMACHINE_SHELL=y` 且 `CONFIG_WASMACHINE_SHELL_CMD_WIFI=y` |
| AP | 已知 SSID 与密码；PC（跑 host_tool 时）与设备同一 AP |

## 分步说明

### 1. 启用 sta 命令（默认 y）

```ini
CONFIG_WASMACHINE_SHELL=y
CONFIG_WASMACHINE_SHELL_CMD_WIFI=y
```

`wm_shell_init()` 会调用 `shell_regitser_cmd_wifi()`，内部初始化 Wi-Fi（`esp_wifi_init` + 默认 STA/AP netif）并注册 `sta` 命令（来源 `components/wasmachine_shell/src/shell_wifi.c`）。

### 2. 连接 AP

```
WASMachine> sta -s myssid -p mypassword
```

命令格式：

```
sta -s <SSID> -p <password>
```

`-s`/`-p` 均可选；若不给，等价于查询/空配置（`shell_wifi.c` 在 SSID 为空时返回 `ESP_FAIL`）。成功后日志（`README.md` §4.5.1）：

```
I (158337) esp_netif_handlers: STA IP: 172.168.30.182, mask: 255.255.255.0, GW: 172.168.30
```

### 3. 之后做远程管理

设备拿到 IP 后即可用 `host_tool`（见 `recipes/app_manager.md`）：

```bash
./host_tool -i app0 -f main/fs_image/wasm/hello_world.wasm -S 172.168.30.182 -P 8080
```

### 4. 与联网 native 的关系

`sta` 只是给固件本身联网。WASM 应用要走 HTTP/MQTT，还需：
- `CONFIG_WASMACHINE_WASM_EXT_NATIVE_HTTP_CLIENT=y` / `CONFIG_WASMACHINE_WASM_EXT_NATIVE_MQTT=y`（见 `recipes/native_http_mqtt.md`）。

### 5. 替代：example_connect

如果不用 shell 配网，也可以在固件里直接调 `example_connect()`（`wm_main.c` 中当 `!CONFIG_WASMACHINE_SHELL_CMD_WIFI` 且 `CONFIG_EXAMPLE_CONNECT_*` 时调用）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `SSID of target AP is shorter` | SSID 为空 | 必须给 `-s <SSID>` |
| `SSID/Password of target AP is longer` | 超出 `wifi_config.sta.ssid` 长度 | SSID/密码长度需 ≤ 32/64 |
| 连不上 / 拿不到 IP | 信号弱 / 密码错 / AP 隔离开启 | 检查密码；关闭 AP 客户端隔离 |
| `host_tool` 仍连不上 | PC 与设备不同 AP / 端口错 | 同网段；`-P` = `CONFIG_WASMACHINE_TCP_PORT` |
| `sta` 命令不存在 | 命令未启用 | 确认 `CONFIG_WASMACHINE_SHELL_CMD_WIFI=y` |

## 参考

- `components/wasmachine_shell/src/shell_wifi.c` — `sta_func` / `shell_regitser_cmd_wifi`
- `README.md` §3.1.4、§4.5.1 — `sta` 与联网
- `recipes/app_manager.md` — 远程安装/查询/卸载
- `recipes/native_http_mqtt.md` — WASM 联网 native
