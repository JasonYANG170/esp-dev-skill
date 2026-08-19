# 设备控制台（matter shell）

> **适用摘要**: 使用 esp-matter 设备端 `matter` shell 命令进行 BLE/Wi-Fi 控制、属性读写、factory reset、桥接设备增删。需 `CONFIG_ENABLE_CHIP_SHELL=y`（示例默认开启）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "matter 控制台命令"
- "设备端 shell"
- "matter ble start"
- "matter esp attribute get/set"
- "matter esp bridge"

## 前置条件

| 条件 | 要求 |
|---|---|
| 配置 | `CONFIG_ENABLE_CHIP_SHELL=y`（生产可关以省 flash） |
| 接入 | 串口 monitor（`idf.py monitor`） |

## 分步说明

> 所有命令前缀为 `matter`。下面命令取自 `developing.rst` 的 Device console 段落。

### 1. BLE 广播控制

```text
matter ble start
matter ble stop
matter ble state
```

### 2. Wi-Fi 模式

```text
matter wifi mode disable
matter wifi mode ap
matter wifi mode sta
matter esp wifi connect <ssid> <password>
```

### 3. 设备静态配置 / 入网码

```text
matter config
matter onboardingcodes
```

### 4. 属性读写（ID 均为十六进制）

```text
matter esp attribute get <endpoint_id> <cluster_id> <attribute_id>
matter esp attribute set <endpoint_id> <cluster_id> <attribute_id> <attribute_value>
```

示例：OnOff cluster（`0x6`）的 `on_off` 属性（`0x0`）在 endpoint 1：

```text
matter esp attribute get 0x1 0x6 0x0
matter esp attribute set 0x1 0x6 0x0 1
```

### 5. 诊断

```text
matter esp diagnostics mem-dump
```

### 6. Factory reset

```text
matter esp factoryreset
```

### 7. Thread / OpenThread（esp32h2/esp32c6/esp32c5）

```text
matter esp ot_cli state
matter esp ot_cli <任意 openthread cli 命令>
```

### 8. 桥接设备增删（aggregator parent）

```text
matter esp bridge add <parent_endpoint_id> <device_type_id>
```

`parent_endpoint_id` 必须是 aggregator device type 的 endpoint。

## 在代码里注册 console 命令

来自 `examples/light/main/app_main.cpp`：

```cpp
#if CONFIG_ENABLE_CHIP_SHELL
    esp_matter::console::diagnostics_register_commands();
    esp_matter::console::wifi_register_commands();
    esp_matter::console::factoryreset_register_commands();
    esp_matter::console::attribute_register_commands();
#if CONFIG_OPENTHREAD_CLI
    esp_matter::console::otcli_register_commands();
#endif
    esp_matter::console::init();
#endif
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 命令不识别 | `CONFIG_ENABLE_CHIP_SHELL=n` | menuconfig 打开，或加 `otcli` 对应的 `CONFIG_OPENTHREAD_CLI` |
| 属性 set 不生效 | endpoint/cluster/attr ID 写成十进制 | 全部用十六进制 |
| `bridge add` 报错 | parent 不是 aggregator | 先创建 aggregator endpoint |
| 生产 flash 太满 | shell 占用 ~54KB flash | 关 `CONFIG_ENABLE_CHIP_SHELL` |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/docs/en/developing.rst` — Device console
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter_console/`（console 组件）
- `D:/esp-skill/espressif-repos/esp-matter/docs/en/optimizations.rst`（关闭 chip-shell 的优化收益）
- `D:/esp-skill/espressif-repos/esp-matter/examples/light/main/app_main.cpp`（console 注册代码）
