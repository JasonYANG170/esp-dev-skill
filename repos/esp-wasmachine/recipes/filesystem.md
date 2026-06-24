# littleFS 文件系统与 WASM 应用打包

> **适用摘要**: 把 `.wasm` 文件放进 `main/fs_image/`，编译生成并烧录 `storage` 分区的 littleFS 镜像，让 `iwasm`/`install` 能读到。

## 触发意图

- "怎么往板子放 wasm 文件"
- "main/fs_image 是什么"
- "idf.py storage-flash"
- "替换 hello_world.wasm"

## 前置条件

| 条件 | 要求 |
|---|---|
| 工程 | `examples/wasmachine/` 拷贝（见 `recipes/new_project.md`） |
| 分区 | `partitions.*.csv` 中存在 `storage` 分区，label 为 `storage` |

## 分步说明

### 1. 文件系统挂载点

`main/wm_main.c` 的 `fs_init()`：

```c
esp_vfs_littlefs_conf_t conf = {
    .base_path = WM_FILE_SYSTEM_BASE_PATH,   /* "/storage" */
    .partition_label = "storage",
    .format_if_mount_failed = false,
    .dont_mount = false,
};
ESP_ERROR_CHECK(esp_vfs_littlefs_register(&conf));
```

`WM_FILE_SYSTEM_BASE_PATH` 来自 `CONFIG_WASMACHINE_FILE_SYSTEM_BASE_PATH`（默认 `/storage`，见 `wm_config.h`）。

### 2. 镜像来源目录

`main/CMakeLists.txt`：

```cmake
littlefs_create_partition_image(storage fs_image)
```

即：`main/fs_image/` 下的所有子目录与文件会被打包进 `storage` 分区镜像。仓库自带：

```
main/fs_image/wasm/hello_world.wasm
```

### 3. 加入自己的 WASM 应用

```bash
# 把编译好的 .wasm 放进 wasm 子目录
cp ~/my_app.wasm main/fs_image/wasm/
```

编译并烧文件系统镜像：

```bash
idf.py build
idf.py storage-flash
```

烧完后控制台：

```
WASMachine> ls wasm
hello_world.wasm
my_app.wasm
WASMachine> iwasm wasm/my_app.wasm -s 262144 -h 262144
```

### 4. 开启 littleFS 兼容选项（参考工程已开）

`examples/wasmachine/sdkconfig.defaults`：

```ini
CONFIG_LITTLEFS_FCNTL_GET_PATH=y
CONFIG_LITTLEFS_OPEN_DIR=y
CONFIG_LITTLEFS_SPIFFS_COMPAT=y
```

### 5. 运行期由 WASM 应用读写文件

libc native 的 `open/read/write/close`（`CONFIG_WASMACHINE_WASM_EXT_NATIVE_LIBC=y`，默认开）会落到该 VFS，路径以 `WM_FILE_SYSTEM_BASE_PATH` 为根。WASM 应用里写相对 `/storage/...` 的绝对路径即可。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 启动挂载失败 | 没烧 `storage` 镜像 | `idf.py storage-flash` |
| 重烧后旧数据丢失 | 设计如此，重烧会清空 | 这是预期行为（README §4.2） |
| `ls wasm` 为空 | `.wasm` 未放进 `main/fs_image/wasm/` | 放入后重新 `build` + `storage-flash` |
| WASM 应用 `open` 失败 | 路径未以 `/storage` 为根 | 用绝对路径 `/storage/...`，或确认 libc native 已开 |
| `storage` 分区找不到 | 分区表缺该行 / label 不是 `storage` | 分区 CSV 必须有 label `storage` 的 `data, spiffs` 行 |

## 参考

- `examples/wasmachine/main/CMakeLists.txt` — `littlefs_create_partition_image(storage fs_image)`
- `examples/wasmachine/main/wm_main.c` — `fs_init()`
- `README.md` §4.2 — 烧文件系统
- `components/wasmachine_core/include/wm_config.h` — `WM_FILE_SYSTEM_BASE_PATH`
- `recipes/native_libc.md` — WASM 应用文件 I/O
