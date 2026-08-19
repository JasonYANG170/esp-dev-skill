# 项目集成与 Kconfig 配置

> **适用摘要**: 把 esp-dsp 作为 ESP-IDF 组件加入新项目或已有项目，选择优化级别与最大 FFT 长度，跑通最小可工作 `app_main`。

> Evidence: `repos/esp-dsp/resources/`, source/examples in `repos/esp-dsp/`, and this recipe path `repos/esp-dsp/recipes/project_setup.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "怎么用 esp-dsp"
- "把 esp-dsp 加到项目"
- "ESP-DSP 安装 / 依赖"
- "menuconfig 配置 DSP"
- "ANSI 还是 Optimized"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | ≥ 4.2（已设置好环境变量） |
| 目标芯片 | ESP32 / ESP32-S3 / ESP32-P4（优化），其它目标走 ANSI |
| 网络 | 首次添加依赖需要访问 IDF Component Registry |

## 分步说明

### 方式一：从官方示例创建全新项目

```bash
# 选定目标芯片（决定优化路径是否生效）
idf.py set-target esp32s3

# 从 component manager 拉取示例并生成工程骨架
idf.py create-project-from-example "espressif/esp-dsp:fft"
# 可用示例名：basic_math conv2d dotprod fft fft4real fft_window fir iir kalman matrix

idf.py build
idf.py -p PORT flash monitor
```

### 方式二：在已有项目中添加 esp-dsp 依赖

```bash
# 在项目根目录执行，自动写入 main/idf_component.yml
idf.py add-dependency "espressif/esp-dsp"
```

或手动编辑 `main/idf_component.yml`：

```yaml
dependencies:
  espressif/esp-dsp: "^1.5.2"
```

随后在源码中包含聚合头并调用 API：

```c
/* main/main.c —— 最小可运行：初始化 FFT 并测量周期 */
#include "esp_log.h"
#include "esp_dsp.h"

static const char *TAG = "main";
#define N_SAMPLES 1024
__attribute__((aligned(16))) static float y_cf[N_SAMPLES * 2];

void app_main(void)
{
    /* esp-dsp 唯一需要 init 的模块是 FFT。
       第一个参数 NULL 表示让库内部分配 sin/cos 表，大小由 Kconfig 决定 */
    esp_err_t ret = dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "FFT init failed: %i", ret);
        return;
    }
    /* ... 填充 y_cf 后调用 dsps_fft2r_fc32 / dsps_bit_rev_fc32 ... */
    ESP_LOGI(TAG, "esp-dsp ready, CONFIG_DSP_MAX_FFT_SIZE=%d", CONFIG_DSP_MAX_FFT_SIZE);
}
```

### 方式三：git clone 后作为 extra component

```bash
git clone https://github.com/espressif/esp-dsp.git components/esp-dsp
idf.py reconfigure
```

`main/CMakeLists.txt` 只需：

```cmake
idf_component_register(SRCS "main.c")
```

### 选择优化级别与最大 FFT 长度

```bash
idf.py menuconfig
# Component config  --->
#   DSP Library  --->
#     DSP Optimization (Optimized)  --->
#       (X) Optimized   # 仅 ESP32/S3/P4/S31 可选；其它目标强制 ANSI
#       ( ) ANSI C
#     Maximum FFT length (4096)  --->
#       512 / 1024 / 2048 / 4096 / 8192 / 16384 / 32768
```

- **Optimized**（默认，仅受支持目标）：宏自动映射到 `_aes3`/`_ae32`/`_arp4`。
- **ANSI C**：纯 C 实现，用于移植到非优化目标或与优化版做交叉验证。
- **Maximum FFT length**：决定内部 sin/cos 表大小，必须 ≥ 你要调用的最大 FFT 点数；默认 4096。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `undefined reference to dsps_fft2r_fc32` | 未添加依赖 / 组件未被发现 | `idf.py add-dependency "espressif/esp-dsp"`，或确认 `components/esp-dsp` 存在 |
| 编译报 `CONFIG_DSP_MAX_FFT_SIZE` 未定义 | 未 include `sdkconfig.h`（间接由 fft 头引入） | 重新 `idf.py reconfigure`，或确认已 `set-target` |
| `ESP_ERR_DSP_PARAM_OUTOFRANGE` | FFT 点数超过 Kconfig 设置 | 在 menuconfig 调大 Maximum FFT length |
| 优化版未生效（周期数高） | 目标芯片非 ESP32/S3/P4 | 该目标仅支持 ANSI；或确认 `set-target` 正确 |
| `fatal error: esp_dsp.h: No such file` | 组件未注册 / 未 reconfigure | `idf.py reconfigure`，检查 `managed_components/` 是否拉到 |

## 参考

- `examples/fft/README.md` — 含 menuconfig 选择优化/ANSI 的官方说明
- `examples/fft/main/CMakeLists.txt` — `idf_component_register(SRCS "dsps_fft_main.c")`
- 仓库根 `README.md` — `idf.py add-dependency` 与 `create-project-from-example` 用法
- `Kconfig` — `DSP_OPTIMIZATION` choice 与 `DSP_MAX_FFT_SIZE_*` choice 定义
