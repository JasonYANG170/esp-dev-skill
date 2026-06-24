# ESP-AT 仓库 examples 索引

> 所有路径均为 esp-at 仓库内的真实示例（`examples/`）。描述综合自各示例 README。

| 路径 | 类别 | 说明 |
|---|---|---|
| `examples/at_custom_cmd/` | 自定义指令 | 以组件方式在不改 esp-at 源码的前提下添加用户自定义 AT 指令；含 `custom/at_custom_cmd.c`（`+TEST` 四类型示例）、`include/at_custom_cmd.h`、`CMakeLists.txt`。要求 esp-at 2024-02 之后快照。 |
| `examples/at_override_module_config/` | 模块配置覆盖 | 外部目录覆盖默认模块配置（sdkconfig.defaults、patch、at_customize.csv、factory_param_data.csv、ble_data），不改 esp-at 源码。要求 2024-03 之后快照。 |
| `examples/at_storage_security/` | 安全存储 | 用 `__wrap_nvs_set_blob/str`、`__wrap_nvs_get_blob/str` 函数包裹拦截 NVS 读写，AES-CTR 加密敏感数据（如含 "key" 的键、Wi-Fi 密码）；自定义指令 `AT+STORAGE_NVS_KEY` 管理密钥。 |
| `examples/at_interface_security/` | 接口安全 | ESP 设备与主机 MCU 间 AT 指令 AES-CTR 加解密安全通信；附 Python 脚本 `at_intf_security_host.py` 模拟主机 MCU。 |
| `examples/at_fs_to_http_server/` | FS+HTTP | 从文件系统读取文件经 HTTP POST 上传到服务器；指令 `AT+FS_TO_HTTP_SERVER=<"dst_path">,<url_len>`。需 `AT_FS_COMMAND_SUPPORT`。 |
| `examples/at_http_get_to_fs/` | HTTP+FS | HTTP GET 下载文件存到文件系统；指令 `AT+HTTPGET_TO_FS=<"dst_path">,<url_len>[,<timeout_ms>]`。需 `AT_FS_COMMAND_SUPPORT`。 |
| `examples/at_spi_master/` | SPI 接口 | SPI AT 主机端示例，含 `spi/`（SPI 模式，支持 ESP32-C 系列，SPI slave ≤10M）与 `sdspi/`（SDIO SPI 模式，≤40M）。ESP32 SPI 不再支持，改用 SDIO SPI。 |
| `examples/at_sdio_host/` | SDIO 接口 | SDIO AT 主机端示例，MCU 经 SDIO 协议与作为 Slave 的 ESP32 通信；含 `ESP32/` 与 `STM32/` 平台子目录、`res/`（数据流图等）。 |

## 使用模式（共同）

所有示例都通过 `AT_CUSTOM_COMPONENTS` 环境变量纳入 esp-at 构建：

```bash
# Linux/macOS
export AT_CUSTOM_COMPONENTS="/abs/path/to/example_dir"

# Windows
set AT_CUSTOM_COMPONENTS=C:\abs\path\to\example_dir

# 多组件用空格分隔
export AT_CUSTOM_COMPONENTS="/path1 /path2"
```

或写入 `build.py` 的 `setup_env_variables()` 函数（适合网页编译）。

## 关键源码位置（非 example，但常用作参考）

| 路径 | 说明 |
|---|---|
| `main/app_main.c` | 固定入口（`esp_at_init()` 调用链） |
| `main/Kconfig` | 全部 AT 功能 Kconfig 选项 |
| `main/interface/{uart,spi,sdio,socket}/` | 四种通信接口实现 |
| `components/at/include/` | 公开 API 头文件（esp_at.h / esp_at_core.h 等） |
| `components/at/lib/` | 预编译 AT 核心库（不开源） |
| `components/customized_partitions/raw_data/factory_param/factory_param_data.csv` | 工厂参数（模块/引脚/国家码等） |
| `components/customized_partitions/raw_data/ble_data/gatts_data.csv` | BLE 服务定义 |
| `components/fs_image/index.html` | Web Server 默认页面 |
| `module_config/module_<name>/` | 各模块配置（sdkconfig.defaults、at_customize.csv、patch 等） |
| `tools/at.py` | 修改打包 factory bin 的 Python 工具 |
| `build.py` | 封装 idf.py 的构建脚本 |
