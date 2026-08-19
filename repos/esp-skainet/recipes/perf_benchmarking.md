# 性能基准测试与测试报告生成

> **适用摘要**: 用 `perf_tester` 控制台与 `test/` 测试工程量化 WakeNet/MultiNet 的唤醒率（RAR）、误唤醒率（FAR）与 CPU/内存占用，并生成 pass/fail 报告。适用于产品发布前的精度验收与回归测试。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-skainet/resources/`, source/examples in `repos/esp-skainet/`, and this recipe path `repos/esp-skainet/recipes/perf_benchmarking.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "测试唤醒率"
- "WakeNet 性能基准"
- "误唤醒率 FAR"
- "RAR 测试 / 响应准确率"
- "perf_tester 控制台"
- "生成测试报告 / CI 跑 wakenet"

## 前置条件

| 条件 | 要求 |
|---|---|
| 测试工程 | `test/wakenet/`（唤醒）/ `test/multinet/`（命令词） |
| 依赖组件 | `components/perf_tester/`（提供 `offline_wn_tester_start` + 控制台命令） |
| SD 卡 | FAT32，根目录放 `{wn_name}.csv` 与 WAV 测试集；FAR 另需 `far_48h.csv` |
| Kconfig | `SR_WN_WN9_*`（被测唤醒词）；CI 走 `sdkconfig.ci.<wn_name>` |
| 测试音频 | 16 kHz、16-bit、3 通道 WAV `[mic1, mic2, ref]`（`TESTER_WAV_3CH`） |

## 分步说明

### 1. 三种测试入口

| 入口 | 用途 | 位置 |
|---|---|---|
| `perf_tester` 控制台（`config`/`start`） | 交互式快速验证 CPU/内存/触发次数 | `components/perf_tester/` |
| 串口命令 `rar` / `far` | 离线跑 SD 卡上的 CSV 测试集，生成报告 | `test/wakenet/main/wakenet_main.c` |
| pytest 脚本 | CI 自动化，解析串口日志生成 `report.json` | `test/wakenet/pytest_wakenet.py` |

### 2. perf_tester 控制台命令（交互式）

烧录任意集成了 `perf_tester` 组件的工程后，串口会出现 `perf_tester>` 提示符（来自 `wakenet_main.c` 的 `repl_config.prompt`）：

```
perf_tester> help
perf_tester> config fast pink 5      # 模式=fast / 噪声=pink / SNR=5dB
perf_tester> start                   # 开始测试
```

`config` 三参数（定义于 `perf_tester_cmd.c` 的 `argtable`）：

| 参数 | 取值 | 含义 |
|---|---|---|
| mode | `fast` / `norm` | fast：快速跑全部用例；norm：完整跑 |
| noise | `all` / `none` / `pink` / `pub` | 按文件名里的噪声类型过滤（`check_noise` 匹配 `pink`/`pub`/`silence` 子串） |
| snr | `all` / `none` / `0` / `5` / `10` | 按文件名里两个 `dB` 段相减得到的 SNR 过滤（`check_snr`） |

> 默认值（`get_perf_tester_config`）：`mode=fast`、`noise=all`、`snr=all`。

### 3. 准备 RAR 测试集（唤醒率）

`{wn_name}.csv` 放 SD 卡根目录，`wn_name` 必须与 menuconfig 选的唤醒词名一致（如 `wn9_hilexin`）：

```csv
filename,required,total
/sdcard/hilexin_0dB_silence.wav,285,300
/sdcard/hilexin_0dB_pub_-10dB.wav,270,300
/sdcard/hilexin_0dB_pink_-10dB.wav,270,300
/sdcard/hilexin_5dB_pink_5dB.wav,260,300
```

| 列 | 含义 |
|---|---|
| filename | SD 卡上 WAV 全路径（16k/16bit/3ch） |
| required | 通过所需的最少触发次数（通常 total 的 85%~95%） |
| total | 该文件里真实唤醒词总数（ground truth） |

> WAV 文件名编码了测试条件：`<wn>_<clean>dB_<noise>_<noise>dB.wav`，SNR = 第一个 dB − 第二个 dB。`check_snr` 依赖该命名。

### 4. 准备 FAR 测试集（误唤醒率）

标准 48 小时连续音频（`far_48h.csv`），内容构成：

| 类型 | 时长 |
|---|---|
| 中文语音 | 22 小时 |
| 英文语音 | 22 小时 |
| 音乐 | 4 小时 |

`far_48h.csv` 同样放 SD 卡根目录；文件清单由 `read_csv_file` 解析（结构与 RAR 一致，`required` 通常为 0）。

### 5. 烧录并运行（test/wakenet）

```bash
cd test/wakenet
idf.py set-target esp32s3
idf.py menuconfig          # Audio Media HAL 选板；选 SR_WN_WN9_HILEXIN 等
idf.py flash monitor
```

进入 `perf_tester>` 后：

```
perf_tester> far            # 先跑 FAR（自动开 debug 模式，打印 detection threshold）
perf_tester> rar            # 再跑 RAR
```

`wakenet_main.c` 里 `start_far_test` / `start_rar_test` 的关键调用：

```c
// test/wakenet/main/wakenet_main.c
static int start_rar_test(int argc, char **argv)
{
    srmodel_list_t *models = esp_srmodel_init("model");
    char *wn_name = esp_srmodel_filter(models, ESP_WN_PREFIX, NULL);
    char csv_file[128];
    char log_file[128];
    sprintf(csv_file, "/sdcard/%s.csv", wn_name);     // 如 /sdcard/wn9_hilexin.csv
    sprintf(log_file, "/sdcard/%s.log", wn_name);

    esp_sr_set_debug_mode(1);                          // 开 debug 输出阈值
    wakenet_test(models, csv_file, log_file);
    return 0;
}
```

`wakenet_test` 内部用 `"MMR"` 输入格式、关 NS、调 `offline_wn_tester_start`：

```c
static void* wakenet_test(srmodel_list_t *models, const char *csv_file, const char* log_file)
{
    afe_config_t *afe_config = afe_config_init("MMR", models, AFE_TYPE_SR, AFE_MODE_LOW_COST);
    afe_config->ns_init = false;
    afe_config->ns_model_name = NULL;
    perf_tester_config_t *tester_config = get_perf_tester_config();
    void *task_handle = offline_wn_tester_start(csv_file, log_file, NULL, afe_config,
                                                TESTER_WAV_3CH, tester_config);
    afe_config_free(afe_config);
    return task_handle;
}
```

### 6. perf_tester 核心 API（components/perf_tester）

```c
#include "wn_perf_tester.h"      // offline_wn_tester_start / _stop
#include "perf_tester_cmd.h"     // perf_tester_config_t / register_* / check_noise / check_snr

// 输入音频类型枚举（wn_perf_tester.h）
typedef enum {
    TESTER_PCM_3CH = 0,   // 3 通道 PCM [mic1, mic2, ref, ...]
    TESTER_WAV_3CH = 1,   // 3 通道 WAV（带 WAV 头，自动解码）
    TESTER_PCM_1CH = 2,   // 单通道 PCM
    TESTER_WAV_1CH = 3,   // 单通道 WAV
} tester_audio_t;

// 启动离线唤醒测试（创建 feed/fetch 任务，跑完自动打印报告）
void* offline_wn_tester_start(const char *csv_file,
                              const char *log_file,
                              const esp_afe_sr_iface_t *afe_handle,  // NULL 时内部从 config 取
                              afe_config_t *afe_config,
                              int audio_type,                         // TESTER_WAV_3CH 等
                              perf_tester_config_t *config);

void  offline_wn_tester_stop(void *tester);
```

`perf_tester_config_t` 结构（`perf_tester_cmd.h`）：

```c
typedef struct {
    char mode[32];    // "fast" / "norm"
    char noise[32];   // "all" / "none" / "pink" / "pub"
    char snr[32];     // "all" / "none" / "0" / "5" / "10"
    int  flag;        // update flag
} perf_tester_config_t;
```

### 7. 串口报告格式（`print_wn_report`，来自 wn_perf_tester.c）

跑完每个文件及全部完成后，串口会输出以下格式的报告（pytest 脚本就解析这些行）：

```
Number of files: 6
Tester PSRAM: 124 KB
Tester SRAM: 8 KB
AFE CPU: 23%
AFE PSRAM: 320 KB
AFE SRAM: 24 KB
File0: /sdcard/hilexin_0dB_silence.wav
File0, trigger times: 295
File0, required times: 285
File0, truth times: 300
File1: ...
Total trigger times: 1620
Total required times: 1560
Total truth times: 1800
TEST DONE
```

> 判定规则：每个 `FileN, trigger times >= required times` 即该用例 PASS。pytest 里对应 `assert trigger_times >= required_times`。

### 8. 调阈值（FAR→RAR 闭环）

FAR 测试默认开 `esp_sr_set_debug_mode(1)`，会打印 detection threshold。流程：

1. 先跑 `far`，观察误触发频次与 debug 打印的阈值分布。
2. 用 `afe_handle->set_wakenet_threshold(afe_data, 1, x)` 调整（范围 0.4–0.9999，调高更严）。
3. 重跑 `rar` 验证唤醒率仍达标。
4. 反复迭代直到 FAR 与 RAR 都满足产品要求。

### 9. CI 自动化（pytest）

```bash
pip install -r tools/ci/requirement.pytest.txt
pytest ./test/wakenet --target esp32s3 --config hilexin --noise pink --snr 5
```

报告生成在 `./pytest_log/`。`pytest_wakenet.py` 还断言内存占用上限（来自真实基线）：

```python
assert psram_size < 1120   # AFE PSRAM < 1120 KB
assert sram_size < 32      # AFE 内部 SRAM < 32 KB
assert trigger_times >= required_times   # 每个用例唤醒率达标
```

CI 构建（`tools/ci/build_apps.py`）会按 `sdkconfig.ci.<name>` 生成多个 bin：

```bash
python tools/ci/build_apps.py ./test -t esp32s3 -vv
# 产出 test/wakenet/IDF_VERSION/build_esp32s3_hilexin、build_esp32s3_hiesp
```

### 10. 录制自己的测试集（可选）

用 `test/record_test_set/` 工具合成带噪声/SNR 的测试音频，再由 ESP32 录制到 SD 卡：

```bash
cd test/record_test_set
pip install -r requirement.txt
python create_test_set.py config.yml        # WakeNet 测试集
python create_mn_test_set.py config_mn.yml  # MultiNet 测试集
```

`config.yml` 里 `clean_set`/`noise_set`/`music_set` 配置源音频，`noise_snr`/`playback_snr` 配置 SNR 组合，脚本自动合成并通过喇叭播放 + DUT 录音。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `Number of files: 0` | CSV 不存在或文件名与 wn_name 不匹配 | 确认 `/sdcard/{wn_name}.csv`，wn_name 用 `esp_srmodel_filter(models, ESP_WN_PREFIX, NULL)` 的返回值 |
| WAV 被跳过 | 采样率/通道数不符 | 测试 WAV 必须 16 kHz、3 通道（`TESTER_WAV_3CH`）；`wav_decoder_get_channel != nch` 会 skip |
| SNR 过滤掉全部文件 | 文件名 dB 段格式不对 | 文件名须含两段 `<num>dB`，如 `hilexin_5dB_pink_5dB.wav`；`check_snr` 解析 `_` 分隔 |
| `offline_wn_tester_start` 直接返回 | `file_num == 0` 时提前调 `print_wn_report` | 先确认 CSV 路径与 SD 卡挂载（`esp_sdcard_init("/sdcard", 10)`） |
| `ringbuffer free space is less than 50%` | feed 跟不上 fetch | tester 会自动切 `delay_flag=1` 全速喂；检查 PSRAM/CPU 占用是否超限 |
| FAR 误触发过多 | 阈值偏低 | 用 debug 模式打印的阈值，调高 `set_wakenet_threshold` 后重测 |
| pytest 内存断言失败 | AFE PSRAM > 1120KB 或 SRAM > 32KB | 关闭非必要算法（NS/AGC），或换 `AFE_MEMORY_ALLOC_MORE_PSRAM` |

## 参考项目

- `test/wakenet/main/wakenet_main.c` — `rar`/`far` 命令注册、`wakenet_test`、`app_main` 控制台初始化
- `components/perf_tester/wn_perf_tester.c` — `offline_wn_tester_start`、`print_wn_report`、`read_csv_file`、`wav_feed_task`/`fetch_task`
- `components/perf_tester/wn_perf_tester.h` — `tester_audio_t` 枚举、`offline_wn_tester_start/stop` 签名
- `components/perf_tester/perf_tester_cmd.c` / `perf_tester_cmd.h` — `config` 命令、`perf_tester_config_t`、`check_noise`/`check_snr`、`register_perf_tester_config_cmd`/`register_perf_tester_start_cmd`
- `test/wakenet/pytest_wakenet.py` — CI 解析报告、内存/唤醒率断言、`report.json` 生成
- `test/wakenet/README.md` — RAR/FAR 测试集格式、`rar`/`far` 串口命令流程
- `test/README.md` — CI 构建（`build_apps.py`）与 pytest 运行
- `test/record_test_set/README.md` + `create_test_set.py` / `create_mn_test_set.py` — 测试集合成与录制
- `test/multinet/main/multinet_main.c` — MultiNet 测试（`offline_mn_tester` + `register_perf_tester_start_cmd`）
