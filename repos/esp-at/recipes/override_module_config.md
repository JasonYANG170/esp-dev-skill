# 用 at_override_module_config 覆盖模块配置

> **适用摘要**: 通过 `at_override_module_config` 外部目录覆盖默认模块配置（sdkconfig.defaults、补丁、分区表、工厂参数、ble_data 等），无需修改 esp-at 仓库源码，便于在自有 git 仓库轻量托管定制内容。

> Evidence: `repos/esp-at/resources/`, source/examples in `repos/esp-at/`, and this recipe path `repos/esp-at/recipes/override_module_config.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "覆盖模块配置"
- "不改 esp-at 源码定制固件"
- "at_override_module_config"
- "自定义 sdkconfig.defaults"
- "添加自定义 patch"

## 前置条件

| 条件 | 要求 |
|---|---|
| 主工程 | esp-at 仓库（2024 年 3 月之后任意 commit 快照均可） |
| 覆盖目录 | `at_override_module_config/`（来自 `examples/at_override_module_config`，置于 esp-at 仓库外） |

## 分步说明

`at_override_module_config` 工作方式：用同名文件覆盖原生 `module_config/module_<your_module>/` 下的配置。可覆盖以下五类（按需覆盖一项或多项，未覆盖的沿用原生配置）：

### 1. 覆盖系统配置（sdkconfig.defaults）

原生文件为 `module_config/module_<your_module>/sdkconfig.defaults`（关闭 silence）或 `sdkconfig_silence.defaults`（开启 silence）。在覆盖目录新增：

```
at_override_module_config/sdkconfig.defaults
```

```
# 开启 WebSocket、关闭 mDNS
CONFIG_AT_WS_COMMAND_SUPPORT=y
CONFIG_AT_MDNS_COMMAND_SUPPORT=n
```

构建系统会用它作为系统配置。

### 2. 覆盖补丁目录（patch）

原生目录为 `module_config/module_<your_module>/patch`。复制原生 patch 目录到覆盖目录，新增补丁文件：

```
at_override_module_config/patch/at_example.patch
at_override_module_config/patch/patch_list.ini
```

在 `patch_list.ini` 中指定补丁：

```ini
# at_override_module_config/patch/patch_list.ini
at_example.patch
```

### 3. 覆盖分区表

覆盖 `at_customize.csv`（自定义分区见 `recipes/customize_partitions.md`）。

### 4. 覆盖工厂参数

覆盖 `factory_param_data.csv`（引脚等参数见 `recipes/set_port_pin.md`）。

### 5. 覆盖 BLE 数据

覆盖 `gatts_data.csv`（BLE 服务自定义见 `recipes/customize_ble_service.md`）。

### 构建流程

```bash
# 1. 克隆 esp-at 主工程
git clone --recursive https://github.com/espressif/esp-at.git
cd esp-at

# 2. 把 examples/at_override_module_config 复制到工程外并按需修改
cp -r examples/at_override_module_config /path/to/my_override
# 编辑 /path/to/my_override/sdkconfig.defaults 等

# 3. 设环境变量（可同时指定自定义组件与覆盖目录）
export AT_CUSTOM_COMPONENTS="/path/to/my_override"
# 或同时加载多个：
# export AT_CUSTOM_COMPONENTS="/path/to/my_override /path/to/at_custom_cmd"

# 4. 安装、配置、编译、烧录
./build.py install
./build.py menuconfig
./build.py build
./build.py -p /dev/ttyUSB0 flash
```

> Windows 用 `set AT_CUSTOM_COMPONENTS=...`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 覆盖未生效 | 文件名/目录名与原生不一致 | 必须用相同文件名与目录名覆盖 |
| 同时托管 at_custom_cmd 与 override 冲突 | 环境变量只设了一个 | `AT_CUSTOM_COMPONENTS` 用空格分隔多个路径 |
| patch 应用失败 | 补丁路径在 `patch_list.ini` 未登记 | 在 `patch_list.ini` 列出补丁文件名 |
| sdkconfig.defaults 不生效 | 主菜单已生成 sdkconfig 缓存 | 删除 `build/` 重新 install/build |
| commit 太旧不兼容 | override 要求 2024-03 之后快照 | 更新 esp-at 到较新 commit |

## 参考

- 示例：`examples/at_override_module_config/`（含 README.md 详细说明）
- 自定义组件示例：`examples/at_custom_cmd/`
- 模块配置：`module_config/module_<name>/`
- 工厂参数：`components/customized_partitions/raw_data/factory_param/factory_param_data.csv`
