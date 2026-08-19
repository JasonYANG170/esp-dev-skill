# 编译、烧录与监视

> **适用摘要**: 完成 ESP-WASMachine 固件的 `set-target`、`build`、`storage-flash`、`flash monitor` 全流程，并验证启动日志。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-wasmachine/resources/`, source/examples in `repos/esp-wasmachine/`, and this recipe path `repos/esp-wasmachine/recipes/build_flash.md`.

## 触发意图

- "怎么编译 wasmachine"
- "烧录固件和文件系统"
- "idf.py storage-flash 是什么"
- "看不到 WASMachine> 提示符"

## 前置条件

| 条件 | 要求 |
|---|---|
| 环境 | ESP-IDF v5.1.x–v5.5.x/master，已 `. ./export.sh` |
| 工程 | 已 `set-target` 并配置�� `sdkconfig.defaults`（见 `recipes/new_project.md`） |
| 串口 | 板子通过 USB 连接，`idf.py -p <PORT>` 可识别 |

## 分步说明

### 1. 配置目标（如尚未）

```bash
idf.py set-target esp32s3     # 或 esp32 / esp32c6 / esp32p4
```

### 2. 编译固件

```bash
idf.py build
```

产物：bootloader、partition table、`wasmachine.bin`，以及由 `main/fs_image/` 生成的 `storage` 分区 littleFS 镜像。

### 3. 烧录文件系统镜像（关键）

littleFS 挂载在 `storage` 分区（label `"storage"`），**首次或更新 WASM 应用时必须单独烧**：

```bash
idf.py storage-flash
```

> 注意：重新烧文件系统镜像会**清空**之前 `storage` 分区中的数据（README §4.2）。

### 4. 烧固件并打开监视器

```bash
idf.py flash monitor
```

成功启动后会看到（README §4.3）：

```
Type 'help' to get the list of commands.
Use UP/DOWN arrows to navigate through command history.
Press TAB when typing command name to auto-complete.
WASMachine>
```

### 5. 验证

在 `WASMachine>` 提示符下：

```
ls wasm
iwasm wasm/hello_world.wasm
```

应输出：

```
Hello World!
```

### 6. （可选）改波特率/端口

```bash
idf.py -p /dev/ttyUSB0 -b 921600 flash monitor
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 启动日志报 littleFS 挂载失败 | 没烧 `storage` 镜像 | `idf.py storage-flash` 后再 `flash monitor` |
| `iwasm` 报 "pkg_type=… is not support" | `.wasm` 文件格式不对 | 用 WASM 工具链（如 wasi-sdk）重新编译，确认是合法 bytecode/AOT |
| 分区重叠 / 启动循环 | 分区表与 flash 容量不匹配 | 4MB 用 `partitions.4mb.single_app.csv`，8MB 用 `.8mb.csv` |
| `assert` 后重启 | `wasm_runtime_full_init` 失败，多为堆不足 | S3/P4 开 PSRAM；调大堆或精简 native 扩展 |
| 看不到 `WASMachine>` 提示符 | Shell 未启用 | 确认 `CONFIG_WASMACHINE_SHELL=y` |

## 参考

- `README.md` §4.2、§4.3、§4.4 — 烧录与运行
- `examples/wasmachine/main/CMakeLists.txt` — `littlefs_create_partition_image(storage fs_image)`
- `examples/wasmachine/partitions.*.csv` — 分区表
- `recipes/filesystem.md` — 往文件系统加 `.wasm`
