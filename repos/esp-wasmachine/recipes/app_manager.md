# App Manager 与 host_tool 远程管理

> **适用摘要**: 启用 WAMR App Manager + TCP 服务，用 shell 的 `install`/`uninstall`/`query` 或 Linux 上的 `host_tool` 远程安装、卸载、查询常驻 WASM applet。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-wasmachine/resources/`, source/examples in `repos/esp-wasmachine/`, and this recipe path `repos/esp-wasmachine/recipes/app_manager.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "安装 wasm 应用"
- "host_tool 怎么用"
- "远程管理 wasm applet"
- "install/uninstall/query 命令"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_WASMACHINE_APP_MGR=y`、`CONFIG_WASMACHINE_TCP_SERVER=y`、`CONFIG_WASMACHINE_TCP_PORT`（默认 8080） |
| Shell | `CONFIG_WASMACHINE_SHELL=y` 且 `CONFIG_WASMACHINE_SHELL_CMD_INSTALL/UNINSTALL/QUERY=y` |
| 网络（远程） | 设备已 `sta` 连到 AP，且 PC 与设备在同一 AP 网段 |
| host_tool | 仅 Linux 可编译：`wasm-micro-runtime/test-tools/host-tool` |

## 分步说明

### 1. 在 sdkconfig 中启用 App Manager

```ini
CONFIG_WASMACHINE_APP_MGR=y
CONFIG_WASMACHINE_TCP_SERVER=y
CONFIG_WASMACHINE_TCP_PORT=8080
CONFIG_WASMACHINE_SHELL=y
CONFIG_WASMACHINE_SHELL_CMD_INSTALL=y
CONFIG_WASMACHINE_SHELL_CMD_UNINSTALL=y
CONFIG_WASMACHINE_SHELL_CMD_QUERY=y
```

固件 `app_main` 调用 `wm_wamr_app_mgr_init()` 后会拉起 App Manager 线程与 TCP 服务线程（端口 `CONFIG_WASMACHINE_TCP_PORT`）。

### 2. （仅远程）让设备连 AP

```
WASMachine> sta -s myssid -p mypassword
```

成功后会打印（`README.md` §4.5.1）：

```
I (158337) esp_netif_handlers: STA IP: 172.168.30.182, mask: 255.255.255.0, GW: 172.168.30
```

### 3a. 用 shell 本地安装

来源 `components/wasmachine_shell/src/shell_install.c`：

```
install <file> -i <name> [--heap <bytes>] [--type <type>] [--timer <n>] [--watchdog <ms>]
```

例：

```
WASMachine> install wasm/hello_world.wasm -i app0
```

`install` 内部构造请求 `/applet?name=app0`，`COAP_PUT`，`FMT_APP_RAW_BINARY`，通过 `wm_wamr_app_send_request(..., INSTALL_WASM_APP)` 发送，并轮询 `app_manager_lookup_module_data` 直到出现或超时（`INSTALL_TIMEOUT = 2000ms`）。

### 3b. 用 host_tool 远程安装

```bash
cd wasm-micro-runtime
./test-tools/host-tool/build/host_tool \
    -i app0 \
    -f main/fs_image/wasm/hello_world.wasm \
    -S 172.168.30.182 \
    -P 8080
```

成功响应（`README.md` §4.5.2）：

```
response status 65
```

### 4. 查询

shell：

```
WASMachine> query            # 全部
WASMachine> query -q app0    # 单个
```

输出（`shell_query.c`）：

```json
{
    "num":   1,
    "applet1":  "app0",
    "heap1":   8192
}
```

host_tool：

```bash
./host_tool -q app0 -S 172.168.30.182 -P 8080
# response status 69  + JSON 体
```

### 5. 卸载

shell：

```
WASMachine> uninstall -u app0
```

`uninstall`（`shell_uninstall.c`）发 `COAP_DELETE` / `FMT_ATTR_CONTAINER`，`REQUEST_PACKET`，轮询直到 `app_manager_lookup_module_data` 返回 NULL（`UNISTALL_TIMEOUT = 1000ms`）。

host_tool：

```bash
./host_tool -u app0 -S 172.168.30.182 -P 8080
# response status 66
```

### 6. 构建 host_tool（Linux）

```bash
git clone -b WAMR-2.2.0 https://github.com/espressif/wasm-micro-runtime.git
cd wasm-micro-runtime/test-tools/host-tool
mkdir build && cd build
cmake ..
make
# 产物：./host_tool
```

`host_tool` 常用参数：`-i` 安装、`-u` 卸载、`-q` 查询、`-f <wasm 文件>`、`-S <设备 IP>`、`-P <端口>`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| "App app0 is already installed" | 同名重复安装 | 先 `uninstall -u app0` 或换名 |
| 安装到第 4 个失败 | 默认上限 3 个 applet | 先卸载不用的 |
| `install` 超时（2s） | flash/网络慢 | 重试；或检查 `storage` 分区空间 |
| `host_tool` 连不上 | 设备 IP/端口不对或不同 AP | 确认 `sta` 已连、IP 正确、端口 = `CONFIG_WASMACHINE_TCP_PORT` |
| `host_tool` 无法在 Windows/macOS 编译 | 仅支持 Linux | 在 Linux（或 WSL）中编译 |
| `query` 返回空 | 没有已安装 applet | 先 `install` |

## 参考

- `components/wasmachine_core/src/wm_wamr_app_mgr.c` — `_app_mgr_thread` / `_tcp_server_thread` / `wm_wamr_app_send_request`
- `components/wasmachine_shell/src/shell_install.c` / `shell_uninstall.c` / `shell_query.c`
- `README.md` §3.2、§4.5 — host_tool 与远程安装/查询/卸载
- `resources/lifecycle.md` — “执行模型 B：App Manager”
