# 本地编译与烧录 ESP-AT 固件

> **适用摘要**: 在本地克隆 esp-at 仓库，安装 ESP-IDF 环境，选择目标芯片/模块，配置功能，编译生成 `factory_XXX.bin` 并烧录到设备。

> Evidence: `repos/esp-at/resources/`, source/examples in `repos/esp-at/`, and this recipe path `repos/esp-at/recipes/build_and_flash.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "编译 AT 固件"
- "build esp-at"
- "烧录 ESP-AT"
- "本地构建 AT 固件"
- "build.py install"

## 前置条件

| 条件 | 要求 |
|---|---|
| 操作系统 | Linux（官方推荐）/ macOS / Windows |
| Python | 3.10.0 及以上 |
| 硬件 | 目标 ESP 模块 + USB 串口连接 |
| 磁盘空间 | ~ several GB（ESP-IDF + 工具链） |

## 分步说明

### 步骤 1：克隆仓库

```bash
# 必须带 --recursive，含子模块
cd ~/esp
git clone --recursive https://github.com/espressif/esp-at.git

# 中国大陆可用镜像
git clone --recursive https://jihulab.com/esp-mirror/espressif/esp-at.git
```

> 路径中不能含空格。

### 步骤 2：安装环境

`build.py install` 会自动安装 Python 包、ESP-IDF 仓库与编译器/工具链。

```bash
./build.py install            # Linux/macOS
python build.py install       # Windows
```

首次运行需交互选择三项：

| 选项 | 说明 | 示例 |
|---|---|---|
| Platform name | 来自 `factory_param_data.csv` 的 platform 列 | `PLATFORM_ESP32C3` |
| Module name | 来自 `factory_param_data.csv` 的 module_name 列 | `MINI-1` |
| Enable silence mode | 裁剪日志、减小固件体积（一般选 No） | `0`（No） |

Platform 取值对照（来自 `factory_param_data.csv`）：`PLATFORM_ESP32`、`PLATFORM_ESP32C2`、`PLATFORM_ESP32C3`、`PLATFORM_ESP32C5`、`PLATFORM_ESP32C6`、`PLATFORM_ESP32C61`、`PLATFORM_ESP32S2`。

> `build/module_info.json` 已存在时这三项不再提示；需重新配置时删除该文件。

### 步骤 3：配置（可选）

```bash
./build.py menuconfig
```

进入 `AT` 菜单按需开启/关闭功能（如 `AT_MQTT_COMMAND_SUPPORT`、`AT_HTTP_COMMAND_SUPPORT` 等）。开启 HTTPS 时务必把 `AT_PROCESS_TASK_STACK_SIZE` 调到 4096 以上。

### 步骤 4：编译

```bash
./build.py build
```

输出位于 `build/factory/`，例如 `factory_WROOM-32.bin`。组合后的 factory bin 含 bootloader、partition、应用等。

### 步骤 5：烧录

```bash
./build.py -p /dev/ttyUSB0 flash      # Linux
python build.py -p COM3 flash         # Windows
```

若启动时打印 `ota data partition invalid`，先擦除整片 Flash 再烧：

```bash
./build.py erase_flash
./build.py -p /dev/ttyUSB0 flash
```

### 步骤 6：验证

用串口工具（115200, 8N1）连接命令端口，发送：

```
AT
AT+GMR
```

收到 `OK` 与版本信息即成功。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ota data partition invalid` 无法启动 | OTA 数据分区无效 | `./build.py erase_flash` 后重新 `flash` |
| 固件体积超 OTA 分区 | 开启蓝牙等大功能 | 扩大 OTA 分区或关闭非必要功能 |
| 选错 Platform/Module | `module_info.json` 缓存 | 删除 `build/module_info.json` 后重新 `install` |
| `idf.py` 子命令未找到 | build.py 已封装 idf.py | 用 `./build.py <cmd>` 即可，`./build.py --help` 查看 |
| 路径含空格导致失败 | ESP-AT 不支持空格路径 | 把工程放到无空格目录 |
| silence mode 误开 | 日志被裁剪，调试困难 | 重新 install 选 No，或单独配 `How_to_enable_more_AT_debug_logs` |

## 参考

- 仓库文档：`docs/en/Compile_and_Develop/How_to_clone_project_and_compile_it.rst`
- 构建脚本：`build.py`（工程根目录）
- 工厂参数：`components/customized_partitions/raw_data/factory_param/factory_param_data.csv`
- 下载指南：`docs/en/Get_Started/Downloading_guide.rst`
