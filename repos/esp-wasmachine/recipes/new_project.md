# 新建 WASMachine 固件项目

> **适用摘要**: 从 `examples/wasmachine` 拷贝并配置一个新的 ESP-WASMachine（WebAssembly 虚拟机）固件工程，确定目标芯片、分区表与启用的组件。

## 触发意图

- "新建 WASMachine 项目"
- "创建 esp-wasmachine 工程"
- "ESP32 跑 WASM 怎么搭"
- "wasmachine_core 怎么用"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | v5.1.x–v5.5.x/master 已安装并 `. ./export.sh` |
| 参考工程 | `examples/wasmachine/` |
| 目标板 | ESP32-DevKitC / ESP32-S3-BOX / ESP32-S3-DevKitC / ESP32-C6-DevKitC / ESP32-P4-Function-EV-Board 之一 |

## 分步说明

### 1. 拷贝参考工程

ESP-WASMachine 仓库内只有一个参考固件工程 `examples/wasmachine/`，它已经接好了 `wasmachine_core` 与各扩展组件。新工程直接拷贝它：

```bash
cp -r examples/wasmachine ~/my_wasmachine
cd ~/my_wasmachine
```

工程结构（真实）：

```
my_wasmachine/
├── CMakeLists.txt
├── main/
│   ├── wm_main.c          # app_main，初始化顺序固定
│   ├── CMakeLists.txt      # littlefs_create_partition_image(storage fs_image)
│   ├── idf_component.yml   # 托管依赖
│   └── fs_image/wasm/      # 烧入文件系统的 .wasm 放这里
├── partitions.4mb.single_app.csv / .8mb.csv / .16mb.csv
└── sdkconfig.defaults{,.esp32,.esp32c6,.esp32s3,.esp32p4,.esp-box,.esp32_p4_function_ev_board}
```

### 2. 选择目标芯片并生成 sdkconfig

```bash
# ESP32-DevKitC
idf.py set-target esp32

# ESP32-S3-DevKitC
idf.py set-target esp32s3

# ESP32-C6-DevKitC
idf.py set-target esp32c6

# ESP32-P4（普通 EV 板）
idf.py set-target esp32p4
```

板级覆盖文件通过 `SDKCONFIG_DEFAULTS` 叠加：

```bash
# ESP32-S3-BOX / S3-BOX-Lite
idf.py -DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.esp-box" set-target esp32s3

# ESP32-P4-Function-EV-Board
idf.py -DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.esp32_p4_function_ev_board" set-target esp32p4
```

### 3. 在 `sdkconfig.defaults` 中开关组件

参考 `examples/wasmachine/sdkconfig.defaults`（默认即开启 App Manager + Shell + HTTP/MQTT/RainMaker/Wi-Fi 配网）：

```ini
CONFIG_WASMACHINE_APP_MGR=y
CONFIG_WASMACHINE_SHELL=y
CONFIG_WASMACHINE_WASM_EXT_NATIVE_HTTP_CLIENT=y
CONFIG_WASMACHINE_WASM_EXT_NATIVE_MQTT=y
CONFIG_WASMACHINE_WASM_EXT_NATIVE_WIFI_PROVISIONING=y
# RainMaker 在 P4 上不支持（idf_component.yml 已排除）
```

只想要最小化 VM（一次性 `iwasm` 跑 WASM）：

```ini
CONFIG_WASMACHINE_APP_MGR=n
CONFIG_WASMACHINE_SHELL=y
CONFIG_WASMACHINE_WASM_EXT_NATIVE=y            # libc/libm 默认开
CONFIG_WASMACHINE_WASM_EXT_NATIVE_HTTP_CLIENT=n
CONFIG_WASMACHINE_WASM_EXT_NATIVE_MQTT=n
```

### 4. 确认 `app_main` 初始化顺序

`main/wm_main.c` 的顺序不要改（详见 `resources/lifecycle.md`）：

```c
void app_main(void)
{
    bsp_init();
    fs_init();
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    wm_wamr_init();
#ifdef CONFIG_WASMACHINE_APP_MGR
    wm_wamr_app_mgr_init();
#endif
#ifdef CONFIG_WASMACHINE_SHELL
    wm_shell_init();
#endif
}
```

### 5. 放入 WASM 应用并编译

把你的 `.wasm` 放进 `main/fs_image/wasm/`（仓库自带 `hello_world.wasm`），然后：

```bash
idf.py build
idf.py storage-flash      # 烧 littleFS 文件系统镜像
idf.py flash monitor
```

看到 `WASMachine>` 提示符即成功。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 板子启动后 littleFS 挂载失败 | 没烧文件系统镜像 | 先执行 `idf.py storage-flash` |
| `WASMACHINE_WASM_EXT_NATIVE_MQTT` 在 menuconfig 找不到 | 未开 `WASMACHINE_APP_MGR` | 先 `CONFIG_WASMACHINE_APP_MGR=y`（MQTT 依赖它） |
| `/dev/uart/0` 找不到 | 控制台占用了 UART0 | 改用 USB/USB-Serial-JTAG 控制台，或只用 UART1/2 |
| S3-BOX 屏幕不亮 | 没叠加 `sdkconfig.esp-box` | 用 `-DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.esp-box"` 重新 `set-target` |
| 4MB 板启动失败 | 用了 8MB/16MB 分区表 | 改用 `partitions.4mb.single_app.csv` |

## 参考

- `examples/wasmachine/main/wm_main.c` — `app_main` 初始化顺序
- `examples/wasmachine/sdkconfig.defaults*` — 各目标/板配置
- `components/wasmachine_core/idf_component.yml` — 托管依赖（wasm-micro-runtime `== 2.*`）
- `README.md` §2、§4.1 — 环境与编译
- `resources/config_reference.md` — 全部 Kconfig 选项
