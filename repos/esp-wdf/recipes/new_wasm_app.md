# 创建并编译一个 WASM 应用

> **适用摘要**：从零创建 ESP-WDF WebAssembly 应用工程，配置 `sdkconfig.defaults`、`main/CMakeLists.txt`，编译出 `.wasm`/`.aot` 固件，并通过 `host_tool.py` 安装运行。

## 触发意图

- "新建 WASM 应用"
- "创建 esp-wdf 工程"
- "编译 wasm 固件"
- "如何用 idf.py build 编译"

## 前置条件

| 条件 | 要求 |
|---|---|
| 已安装开发环境 | 在 esp-wdf 根目录运行过 `./install.sh`（Linux/macOS）或 `./install.bat`（Windows） |
| 已 source 环境 | 在根目录执行过 `. ./export.sh`（Windows：`export.bat`） |
| 参考 | `examples/hello_world/`、`examples/simple/timer/` |

## 分步说明

### 1. 工程目录结构（标准 WASI 应用）

```
my_app/
├── CMakeLists.txt
├── sdkconfig.defaults      # 可选，预置关键开关
└── main/
    ├── CMakeLists.txt
    └── my_app.c
```

顶层 `CMakeLists.txt` 只需最小内容（参考 `examples/hello_world/CMakeLists.txt`，构建系统会把 `main/` 注册为组件）：

```cmake
# 顶层 CMakeLists.txt（最小）
# 实际由 esp-wdf 顶层 CMakeLists.txt 驱动，这里可不写或仅 cmake_minimum_required
```

`main/CMakeLists.txt`（直接取自 `examples/hello_world/main/CMakeLists.txt`）：

```cmake
set(srcs
    "my_app.c"
    )

idf_component_register(SRCS ${srcs})
```

### 2. 编写应用（标准 WASI 入口）

```c
/* main/my_app.c —— 标准 WASI 应用，实现 main */
#include <stdio.h>

int main(void)
{
    printf("Hello from my WASM app!\n");
    return 0;
}
```

### 3. 切换到工程目录并配置

```bash
cd esp-wdf
. ./export.sh            # Windows: export.bat
cd examples/hello_world  # 或你自己的工程目录
idf.py menuconfig        # 可选；如已用 sdkconfig.defaults 可跳过
```

> 重要：不要在 esp-wdf 根目录直接 `idf.py build`。顶层 `CMakeLists.txt` 会以 `FATAL_ERROR` 拒绝。

### 4. 编译 WASM 字节码固件

```bash
idf.py build
```

成功输出末尾会出现：

```
[100%] Built target hello_world.wasm
Done
```

产物在 `build/hello_world.wasm`。

### 5. （可选）编译 AOT 固件

需自行构建 `wamrc` 工具（参考 WAMR 文档）。

```bash
idf.py build aot
```

成功输出：

```
Compile success, file hello_world.aot was generated.
Done
```

产物在 `build/hello_world.aot`。

### 6. 用 host_tool.py 安装/管理应用

```bash
# 安装应用（-i），指定服务器地址、端口、堆大小、timer 数等
python tools/host_tool.py -i my_app \
    -S <server_ip> -P <port> \
    -f build/my_app.wasm \
    --heap 102400 --timer 10 --watchdog 10000

# 查询应用信息（-q，不带名称则列出全部）
python tools/host_tool.py -q my_app -S <server_ip> -P <port>

# 卸载应用（-u）
python tools/host_tool.py -u my_app -S <server_ip> -P <port>
```

安装参数：`--type`（应用类型）、`--heap`（堆字节）、`--timer`（最大定时器数）、`--watchdog`（看门狗间隔，毫秒）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `Current directory ... is not buildable` | 在 esp-wdf 根目录执行 `idf.py build` | `cd` 到 `examples/<name>` 或自建工程目录再构建 |
| 找不到 `idf.py` | 未 `source` 环境 | 在根目录执行 `. ./export.sh`（Windows：`export.bat`） |
| `export.sh` 报缺依赖 | 未运行 install 脚本 | 先运行 `./install.sh` / `./install.bat` |
| 产物过大无法安装 | 默认线性内存偏小 | menuconfig 调大 `CONFIG_COMPILER_WASI_INITIAL_MEMORY` |
| AOT 加载失败 | 用了 WASM 固件却以 AOT 方式运行（或反之） | 安装时固件格式与运行模式一致 |

## 参考

- `examples/hello_world/main/hello_world.c`
- `examples/hello_world/main/CMakeLists.txt`
- `tools/host_tool.py`
- `README.md` 第 2–4 节
- `resources/api_reference.md` —— 入口形态说明
- `resources/config_reference.md` —— Kconfig 全表
