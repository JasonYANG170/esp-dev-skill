# 运行时方法（AMETHOD）解耦接口与实现

> **适用摘要**: 用 `AMETHOD(MODULE, METHOD)` 拼装方法名字符串、经 `esp_gmf_element_exe_method` 调用 element，使应用层只依赖方法名而非具体 element 类型，便于在 pool 中替换实现（如 aud_rate_cvt ↔ aud_asrc）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-gmf/resources/`, source/examples in `repos/esp-gmf/`, and this recipe path `repos/esp-gmf/recipes/runtime_methods.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "运行时方法"
- "AMETHOD"
- "exe_method"
- "替换 element 不改上层代码"
- "方法名调用"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `esp_gmf_element.h`、`esp_gmf_method.h`、`esp_gmf_audio_methods_def.h` |
| 参考文档 | `docs/en/gmf-framework/gmf-core/gmf-core-element.rst`（Runtime Methods） |

## 分步说明

### 1. 命名 setter 与 AMETHOD 是同一实现

每个 element 提供完整命名 API（如 `esp_gmf_alc_set_gain`），同时通过 `esp_gmf_audio_methods_def.h` 暴露同名的运行时方法。两种调用方式等价：

```c
// 方式 A：命名 setter
esp_gmf_alc_set_gain(alc_el, 0, -6);

// 方式 B：运行时方法（参数按 args_desc 顺序打包进 buf）
uint8_t buf[2] = { 0 /* idx */, (uint8_t)(-6) /* dB */ };
esp_gmf_element_exe_method(alc_el, AMETHOD(ALC, SET_GAIN), buf, sizeof(buf));
```

### 2. 方法由 element 懒加载

element 在 `ops.load_methods` 里用 `esp_gmf_method_append` 注册方法名、执行函数与参数描述。框架在首次 `esp_gmf_element_get_method` / `esp_gmf_element_exe_method` 时回调并缓存。

```c
// element 内部实现示例（来自文档）
static esp_gmf_err_t set_volume(void *handle, esp_gmf_args_desc_t *args,
                                uint8_t *buf, int len)
{
    uint8_t volume = *buf;
    /* 更新内部状态 */
    return ESP_GMF_ERR_OK;
}

static esp_gmf_err_t load_methods(esp_gmf_element_handle_t handle)
{
    esp_gmf_method_t *methods = NULL;
    esp_gmf_args_desc_t *args = NULL;
    esp_gmf_args_desc_append(&args, "volume", ESP_GMF_ARGS_TYPE_UINT8, sizeof(uint8_t), 0);
    esp_gmf_method_append(&methods, "set_volume", set_volume, args);
    ESP_GMF_ELEMENT_GET(handle)->method = methods;
    return ESP_GMF_ERR_OK;
}
```

### 3. 接口与实现解耦的价值

以采样率转换为例：`aud_rate_cvt` 与硬件 `aud_asrc` 都实现 `RATE_CVT::SET_DEST_RATE`。应用只调方法名：

```c
// 应用层：不关心是软件 rate_cvt 还是硬件 asrc
uint8_t rate_buf[4];
int rate = 16000;
memcpy(rate_buf, &rate, sizeof(rate));
esp_gmf_element_exe_method(rate_or_asrc_el, AMETHOD(RATE_CVT, SET_DEST_RATE),
                           rate_buf, sizeof(rate_buf));
```

替换实现时只需在 pool 注册阶段把 `aud_rate_cvt` 换成 `aud_asrc`（或注册自定义 element 用同名方法），上层调用逻辑零改动。

### 4. 能力描述（caps）配合方法选择

构建流水线时可先查 element 能力（EIGHTCC）挑选，再统一用方法名控制：

```c
esp_gmf_cap_t *caps = NULL;
esp_gmf_element_get_caps(el, &caps);
esp_gmf_cap_t *node = NULL;
esp_gmf_cap_fetch_node(caps, ESP_GMF_CAPS_AUD_RATE_CVT, &node);   // 是否支持采样率转换
esp_gmf_cap_attr_t *attr = NULL;
esp_gmf_cap_find_attr(node, "sample_rate", &attr);                 // 查支持的采样率
```

> 方法名应保持稳定；参数描述告知调用方参数名/类型/大小/偏移。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `exe_method` 返回 NOT_FOUND | 方法名拼错或 element 未 load_methods | 确认 `AMODULE`/`AMETHOD` 大小写与 `esp_gmf_audio_methods_def.h` 一致 |
| 参数不生效 | buf 字节序/大小与 args_desc 不符 | 按 `esp_gmf_args_desc_t` 的 type/size 打包 |
| 替换 element 后方法消失 | 新 element 未实现同名方法 | 自定义替换件需注册相同方法名 |

## 参考

- `docs/en/gmf-framework/gmf-core/gmf-core-element.rst`（Runtime Methods 章节）
- `docs/en/gmf-framework/gmf-elements/gmf-audio.rst`（Runtime Method Call Pattern）
- 头文件 `elements/gmf_audio/include/esp_gmf_audio_methods_def.h`
