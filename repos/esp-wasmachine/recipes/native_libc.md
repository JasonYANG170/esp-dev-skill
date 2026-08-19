# WASM Native libc（文件 I/O 与基础 POSIX）

> **适用摘要**: 启用并理解 WASM libc native API（`open`/`read`/`write`/`ioctl`/`close`/`sleep`/`time`/`rand` 等），WASM 应用通过它访问 VFS 文件与外设。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-wasmachine/resources/`, source/examples in `repos/esp-wasmachine/`, and this recipe path `repos/esp-wasmachine/recipes/native_libc.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "WASM 应用怎么读写文件"
- "libc native API"
- "open/read/write 在 wasm 里能用吗"
- "WASM errno 翻译"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_WASMACHINE_WASM_EXT_NATIVE=y` 且 `CONFIG_WASMACHINE_WASM_EXT_NATIVE_LIBC=y`（均默认 y） |
| 文件系统 | littleFS 已挂载（`recipes/filesystem.md`） |

## 分步说明

### 1. 启用（默认即开）

`components/wasmachine_ext_wasm_native/Kconfig.wasmachine`：

```ini
CONFIG_WASMACHINE_WASM_EXT_NATIVE=y
CONFIG_WASMACHINE_WASM_EXT_NATIVE_LIBC=y
```

### 2. 可用 import 清单

来源 `components/wasmachine_ext_wasm_native/src/wm_ext_wasm_native_libc.c` 的 `wm_libc_wrapper_native_symbol[]`，注册到 `"env"` 模块：

| import | 签名 | 说明 |
|---|---|---|
| `open`   | `($ii)i` | WASM 侧 `O_*` 标志会被重映射（见下） |
| `read`   | `(i*~)i` | |
| `write`  | `(i*~)i` | |
| `pread`  | `(i*~i)i` | |
| `pwrite` | `(i*~i)i` | |
| `lseek`  | `(iIi)I` | offset 与返回值均为 int64 |
| `fcntl`  | `(iii)i` | |
| `fsync`  | `(i)i`   | |
| `close`  | `(i)i`   | |
| `ioctl`  | `(ii*)i` | 仅 `CONFIG_WASMACHINE_EXT_VFS` 时注册 |
| `fstat`  | `(i*)i`  | **不支持**，返回 -1 并告警 |
| `sleep`  | `(i)i`   | |
| `usleep` | `(i)i`   | |
| `time`   | `(*)I`   | |
| `srand`  | `(i)`    | |
| `rand`   | `()i`    | |
| `localtime_r` | 手动注册 | 返回 WASM 侧 `tp` 偏移 |

### 3. WASM 侧 `open` 的标志位

源码 `wm_ext_wasm_native_libc.c` 把 WASM 的 `WASM_O_*` 重映射到宿主 `O_*`：

```c
#define WASM_O_RDONLY    (1 << 26)
#define WASM_O_WRONLY    (1 << 28)
#define WASM_O_RDWR      (WASM_O_RDONLY | WASM_O_WRONLY)
#define WASM_O_APPEND    (1 << 0)
#define WASM_O_CREAT     (1 << 12)
#define WASM_O_TRUNC     (1 << 15)
#define WASM_O_EXCL      (1 << 14)
#define WASM_O_NONBLOCK  (1 << 2)
#define WASM_O_SYNC      (1 << 4)
#define WASM_O_DIRECTORY (1 << 13)
```

> WASM 应用应使用它自己的 WASI/工具链头里的 `O_*` 常量，不要硬编码这些位。

### 4. errno 翻译

`errno_c2wasm()` 把宿主 errno 映射成 `WASI_E*`（如 `WASI_ENOENT=44`、`WASI_EBADF=8`、`WASI_ENOSPC=51`）。WASM 应用读到的 errno 是 WASI 编码。

### 5. WASM 应用用法示例（伪代码）

```c
/* 在 WASM 应用侧（用 wasi-sdk 编译） */
#include <fcntl.h>
#include <unistd.h>

int fd = open("/storage/wasm/data.txt", O_RDWR | O_CREAT, 0666);
write(fd, "hi", 2);
lseek(fd, 0, SEEK_SET);
char buf[8];
int n = read(fd, buf, sizeof(buf));
close(fd);
```

`/storage` 即 `WM_FILE_SYSTEM_BASE_PATH`。`ioctl` 用于外设（见 `recipes/ext_vfs.md`）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `open` 返回 -1 且 `errno=WASI_ENOENT(44)` | 路径错或文件不在镜像 | 用 `/storage/...` 全路径；`idf.py storage-flash` |
| `fstat` 总返回 -1 | libc wrapper 不支持 | 不要依赖 `fstat` 取大小 |
| `errno` 值看着不对 | 是 WASI 编码，非宿主 errno | 查 `WASI_E*` 表（源码 `wm_ext_wasm_native_libc.c`） |
| WASM 里硬编码 `O_*` 位错位 | 直接写了宿主位值 | 用 WASM 工具链头里的 `O_*` |
| `ioctl` 找不到 import | 未开 `CONFIG_WASMACHINE_EXT_VFS` | 开 Extended VFS（见 `recipes/ext_vfs.md`） |

## 参考

- `components/wasmachine_ext_wasm_native/src/wm_ext_wasm_native_libc.c` — 全部 wrapper 与符号注册
- `components/wasmachine_ext_wasm_native/include/wm_ext_wasm_native_macro.h` — `REG_NATIVE_FUNC` 等
- `resources/api_reference.md` §2 — libc import 表
- `recipes/ext_vfs.md` — `ioctl` 外设访问
