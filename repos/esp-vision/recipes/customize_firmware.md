# 客制化 ESP-VISION 固件

> **适用摘要**: 在不同配置层移除或增加能力：Python 模块（`micropython.cmake`）、图像算法（`boards/<BOARD>/imlib_config.h`）、标准 MicroPython 功能（`mpconfigboard.h`）、冻结 Python（`manifest.py`）、板级服务与可选组件（`board.cmake`）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-vision/resources/`, source/examples in `repos/esp-vision/`, and this recipe path `repos/esp-vision/recipes/customize_firmware.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "裁剪固件体积"
- "关闭 / 开启某个算法"
- "冻结自己的 Python 模块"
- "新增 ESP-VISION Python 模块"
- "客制化板级服务"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考文档 | `docs/zh_CN/how-to/customize-firmware.rst`、`docs/zh_CN/how-to/add-python-module.rst` |
| 环境 | ESP-IDF（已 source export）+ ESP-VISION 仓库（含子模块） |
| 决策 | 变更面向所有板（改根 `micropython.cmake`）还是单板（改 `boards/<BOARD>/`） |

## 分步说明

### 1. 客制化 ESP-VISION Python 模块（`micropython.cmake`）

`ESP_VISION_MODULE_SOURCES` 列表决定暴露给 Python 的 C 绑定。仅适用部分芯片的模块放在对应 `IDF_TARGET` 条件中（H.264 / RTSP 当前仅 `esp32p4`）。

```cmake
set(ESP_VISION_MODULE_SOURCES
    ${ESP_VISION_ROOT}/modules/py_display.c
    ${ESP_VISION_ROOT}/modules/py_image.c
    ${ESP_VISION_ROOT}/modules/py_imageio.c
    ${ESP_VISION_ROOT}/modules/py_helper.c
    ${ESP_VISION_ROOT}/modules/py_sensor.c
)
```

移除源码不会自动移除背后的平台服务或托管组件；改完需检查 `target_sources`、`target_link_libraries`、组件清单和 size 报告。

### 2. 客制化图像算法（`boards/<BOARD>/imlib_config.h`）

```c
#define IMLIB_ENABLE_MEAN
#define IMLIB_ENABLE_GAUSSIAN
#define IMLIB_ENABLE_QRCODES
#define IMLIB_ENABLE_APRILTAGS
```

删除某个 `IMLIB_ENABLE_*` 排除不用的算法；增加受支持的定义启用其他算法。未启用的 Python 方法仍在绑定层中，调用时抛异常。未经许可证审查不应移除 `OMV_NO_GPL`。

### 3. 客制化标准 MicroPython 功能（`boards/<BOARD>/port/mpconfigboard.h`）

```c
#define MICROPY_PY_BLUETOOTH (0)
#define MICROPY_PY_ESPNOW (0)
#define MICROPY_PY_NETWORK_WLAN (1)
```

这些宏可能依赖 ESP-IDF 版本与 SoC 能力宏；关闭后可能还需移除对应 ESP-IDF 配置或托管依赖才能获得可观测的体积缩减。

### 4. 客制化冻结 Python 代码（`boards/<BOARD>/manifest.py`）

```python
freeze("$(PORT_DIR)/modules")
freeze("$(ESP_VISION_ROOT)/modules", "py_inisetup.py")
freeze("$(ESP_VISION_ROOT)/boards/<BOARD>", "board_inisetup.py")
include("$(MPY_DIR)/extmod/asyncio")
```

可用 `freeze()`、`module()`、`package()`、`include()` 增加产品启动代码/库/包；不需要时删对应条目。

### 5. 客制化板级服务与可选组件（`boards/<BOARD>/board.cmake`）

存在 `camera.c` / `display.c` / `sdcard.c` 时 `micropython.cmake` 自动选用。板级开关放在 `board.cmake`：

```cmake
set(ESP_VISION_ENABLE_BARCODE OFF)
```

增加组件时注册其源码/IDF 组件并链接到 `usermod_esp_vision_platform`；移除时一次性清掉源码、头路径、编译定义、链接依赖与清单条目。

### 6. 构建并验证

```bash
idf.py --board <BOARD> reconfigure
idf.py --board <BOARD> build
idf.py --board <BOARD> size
```

REPL 中验证：`help("modules")`、`import <module>`、`help(<module>)`，并跑相关示例，对比修改前后 size 报告。

### 附：新增 Python 模块（`USER_C_MODULES`）

```c
/* SPDX-License-Identifier: Apache-2.0 */
#include "py/runtime.h"

static mp_obj_t foo_hello(void) {
    return mp_obj_new_int(42);
}
static MP_DEFINE_CONST_FUN_OBJ_0(foo_hello_obj, foo_hello);

static const mp_rom_map_elem_t foo_module_globals_table[] = {
    { MP_ROM_QSTR(MP_QSTR___name__), MP_ROM_QSTR(MP_QSTR_foo) },
    { MP_ROM_QSTR(MP_QSTR_hello), MP_ROM_PTR(&foo_hello_obj) },
};
static MP_DEFINE_CONST_DICT(foo_module_globals, foo_module_globals_table);

const mp_obj_module_t mp_module_foo = {
    .base = { &mp_type_module },
    .globals = (mp_obj_dict_t *)&foo_module_globals,
};

MP_REGISTER_MODULE(MP_QSTR_foo, mp_module_foo);
```

然后：在 `micropython.cmake` 的 `ESP_VISION_MODULE_SOURCES` 加入 `py_foo.c`；按需在 `modules/qstrdefs_esp_vision.h` 加 `Q(name)`；写 `stubs/foo.pyi`；加 `docs/.../api-reference/foo.rst`。C++ 模块用 `.cpp`（构建已设 `-std=gnu++2b`，参见 `modules/py_espdl.cpp`）。完整步骤见 `docs/zh_CN/how-to/add-python-module.rst`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 移除模块后链接仍带依赖 | 只删了绑定源 | 同时清理平台服务/组件/链接项 |
| 调用算法抛异常 | `imlib_config.h` 未启用 | 启用对应 `IMLIB_ENABLE_*` |
| 改了配置没生效 | 用了旧构建 | `reconfigure` 后 `build` |
| 关 Python 功能体积没减 | 未清对应 IDF 配置/依赖 | 同步移除托管依赖 |
| qstr 未注册 | 受板级开关控制且扫描不到 | 在 `qstrdefs_esp_vision.h` 加 `Q(name)` |

## 参考

- `docs/zh_CN/how-to/customize-firmware.rst`
- `docs/zh_CN/how-to/add-python-module.rst`
- `docs/zh_CN/how-to/add-board.rst`
- `boards/ESP32_P4X_EYE/imlib_config.h`
- `micropython.cmake`
