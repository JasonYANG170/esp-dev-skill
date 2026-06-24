# 用 iwasm 运行 WASM 应用

> **适用摘要**: 在 `WASMachine>` 控制台用 `iwasm` 从文件系统加载并一次性运行 WASM 应用，设置栈/堆与 WASI 环境变量、目录、地址池。

## 触发意图

- "iwasm 怎么用"
- "运行 wasm 应用"
- "iwasm 指定栈大小"
- "WASM 应用传环境变量"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_WASMACHINE_SHELL=y` 且 `CONFIG_WASMACHINE_SHELL_CMD_IWASM=y` |
| 文件系统 | `.wasm` 已在 `main/fs_image/` 并执行过 `idf.py storage-flash`（见 `recipes/filesystem.md`） |

## 分步说明

### 1. 查看可用文件

```
WASMachine> ls wasm
```

### 2. 基本运行

```
WASMachine> iwasm wasm/hello_world.wasm
```

输出（`README.md` §4.4）：

```
Hello World!
```

### 3. 命令格式与参数

来源 `components/wasmachine_shell/src/shell_iwasm.c` 与 `README.md` §3.1.1：

```
iwasm <file> <args> [配置参数]

  -s / --stack_size : WASM 应用栈大小（字节）
  -h / --heap_size  : WASM 应用堆大小（字节）
  -e / --env        : WASI 环境变量，"key=value"，多个用 "," 分隔（仅 libc WASI）
  -d / --dir        : WASI 允许访问的目录，多个用 "," 分隔（仅 libc WASI）
  -a / --addr-pool  : WASI 允许访问的对端地址 CIDR，多个用 "," 分隔（仅 libc WASI）
```

非 libc-WASI 模式（默认参数）：

```
iwasm wasm/demo.wasm -s 262144 -h 262144
```

libc-WASI 模式（需要 `CONFIG_WAMR_ENABLE_LIBC_WASI`）：

```
iwasm wasm/demo.wasm -s 262144 -h 262144 -e "key1=value1" -a 1.2.3.4/15
```

### 4. 传应用参数

`<args>` 用引号包起来，`iwasm` 内部用 `esp_console_split_argv` 拆分：

```
WASMachine> iwasm wasm/demo.wasm "arg1 arg2 arg3"
```

### 5. 默认栈/堆（未传 `-s`/`-h` 时）

由 `CONFIG_WASMACHINE_SHELL_WASM_APP_STACK_SIZE` / `_HEAP_SIZE` 决定，默认均为 `16384`（字节）。重负载应用务必显式指定，例如 256KB：

```
iwasm wasm/big.wasm -s 262144 -h 262144
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 应用崩溃 / 异常退出 | 默认栈/堆太小（16384） | 用 `-s`/`-h` 显式调大，如 `-s 262144 -h 262144` |
| `-e`/`-d`/`-a` 报未知选项 | 未开 libc WASI | 这些选项仅在 `CONFIG_WAMR_ENABLE_LIBC_WASI != 0` 时注册 |
| "pkg_type=… is not support" | 文件不是合法 WASM bytecode/AOT | 用 WASM 工具链重新编译 |
| `iwasm` 命令不存在 | Shell 或该命令未启用 | 确认 `CONFIG_WASMACHINE_SHELL=y` 且 `CONFIG_WASMACHINE_SHELL_CMD_IWASM=y` |
| `iwasm` 阻塞很久不返回 | 设计如此：`iwasm` 在独立线程 `pthread_join`，等 WASM `main` 返回 | 长期驻留的 applet 改用 App Manager（`recipes/app_manager.md`） |

## 参考

- `components/wasmachine_shell/src/shell_iwasm.c` — `iwasm_main` / `iwasm_main_thread`
- `README.md` §3.1.1、§4.4 — 命令格式与运行示例
- `resources/api_reference.md` §10 — shell 命令表
- `resources/lifecycle.md` — “执行模型 A：iwasm”
