# 运行时调整音频效果（EQ/ALC/Sonic/Fade/DRC/MBC）

> **适用摘要**: 在播放流水线运行过程中，通过 element 命名 setter 或运行时方法（AMETHOD）实时调整均衡、自动增益、变速变调、淡入淡出、动态范围控制等效果。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-gmf/resources/`, source/examples in `repos/esp-gmf/`, and this recipe path `repos/esp-gmf/recipes/audio_effects.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "调 EQ"
- "调音量/增益"
- "变速变调"
- "淡入淡出"
- "运行时改效果参数"

## 前置条件

| 条件 | 要求 |
|---|---|
| 流水线 | 已用 `gmf_loader_setup_audio_effects_default` 注册效果 element |
| 参考示例 | `gmf_examples/basic_examples/pipeline_audio_effects` |

## 分步说明

效果 element 由 `gmf_loader_setup_audio_effects_default` 注册，tag 包括 `aud_eq`/`aud_alc`/`aud_sonic`/`aud_fade`/`aud_drc`/`aud_mbc`/`aud_mixer` 等。先用 `esp_gmf_pipeline_get_el_by_name` 取句柄，再调对应 setter。

### 1. ALC 自动增益（每通道独立，范围 [-64, 63] dB）

```c
#include "esp_gmf_alc.h"

esp_gmf_element_handle_t alc_el = NULL;
esp_gmf_pipeline_get_el_by_name(pipe, "aud_alc", &alc_el);
esp_gmf_alc_set_gain(alc_el, 0, -6);   // 左声道 -6 dB
esp_gmf_alc_set_gain(alc_el, 1, -6);   // 右声道 -6 dB
// 值 < -64 视为静音
```

### 2. Sonic 变速/变调（speed、pitch ∈ [0.5, 2.0]，1.0 不变）

```c
#include "esp_gmf_sonic.h"

esp_gmf_element_handle_t sonic_el = NULL;
esp_gmf_pipeline_get_el_by_name(pipe, "aud_sonic", &sonic_el);
esp_gmf_sonic_set_speed(sonic_el, 0.8f);   // 0.8x 速度
esp_gmf_sonic_set_pitch(sonic_el, 1.2f);   // 升调
```

> time-stretch 是非整数分帧，下游（codec_dev IO）必须按 `valid_size` 写硬件，不能假设固定长度。

### 3. EQ 多段均衡（每个 band 独立配置/使能）

```c
#include "esp_gmf_eq.h"

esp_gmf_element_handle_t eq_el = NULL;
esp_gmf_pipeline_get_el_by_name(pipe, "aud_eq", &eq_el);

esp_ae_eq_filter_para_t para = {
    .filter_type = ESP_AE_EQ_FILTER_PEAK,
    .fc          = 1000,
    .q           = 1.0f,
    .gain        = 6.0f,
};
esp_gmf_eq_set_para(eq_el, 0, &para);       // band 0
esp_gmf_eq_enable_filter(eq_el, 0, true);
// filter_num=0 时 open 用内置 10 段默认（31 Hz–16 kHz）
```

### 4. Fade 淡入淡出（可在切轨时 reset 进度）

```c
#include "esp_gmf_fade.h"

esp_gmf_element_handle_t fade_el = NULL;
esp_gmf_pipeline_get_el_by_name(pipe, "aud_fade", &fade_el);
esp_gmf_fade_set_mode(fade_el, ESP_GMF_FADE_MODE_FADE_IN);   // FADE_IN / FADE_OUT
// 切歌复用同一 fade element：重置进度
esp_gmf_fade_reset(fade_el);
```

### 5. 采样率/通道/位深转换（setter 触发 need_reopen）

```c
#include "esp_gmf_rate_cvt.h"

esp_gmf_element_handle_t rate_el = NULL;
esp_gmf_pipeline_get_el_by_name(pipe, "aud_rate_cvt", &rate_el);
esp_gmf_rate_cvt_set_dest_rate(rate_el, 16000);
// setter 触发 need_reopen，下一个 process 边界自动重建 handle（会有短暂停顿）
// 如需平滑切换，先 pause 再改
```

### 6. 用运行时方法（AMETHOD）解耦接口与实现

效果 element 通过 `esp_gmf_audio_methods_def.h` 暴露方法名，应用只依赖方法名不依赖具体 element。例如把 `aud_rate_cvt` 换成硬件 `aud_asrc`，应用代码不变。

```c
#include "esp_gmf_element.h"
// AMETHOD / 方法名宏来自 esp_gmf_audio_methods_def.h

uint8_t buf[2] = { 0 /* channel idx */, (uint8_t)(-6) /* dB */ };
esp_gmf_element_exe_method(alc_el, AMETHOD(ALC, SET_GAIN), buf, sizeof(buf));
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| setter 不生效 | element 未注册或 tag 拼错 | 用 `ESP_GMF_POOL_SHOW_ITEMS` 确认 tag，用 `get_el_by_name` 取正确句柄 |
| 切换采样率后有短暂停顿 | `set_dest_rate` 触发 need_reopen 重建 handle | 平滑切换：先 `pause` 再改 |
| aud_sonic speed=2 输出非半长 | time-stretch 非整数分帧 | 下游按 `valid_size` 处理 |
| mixer 主轨无数据时其他轨阻塞 | 设计为主轨阻塞 0、其余最大延迟 | 想让副轨接管就把它连到输入 port 0 |

## 参考

- `gmf_examples/basic_examples/pipeline_audio_effects/main/pipeline_audio_effects.c`
- `docs/en/gmf-framework/gmf-elements/gmf-audio.rst`（Audio Effects / Format Conversion 章节）
