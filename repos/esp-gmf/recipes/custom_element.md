# 自定义音频 Element 模板

> **适用摘要**: 实现一个自定义 audio element（含 open/process/close 生命周期、输入输出端口属性、acquire/release 数据协议），并注册进 pool 参与 pipeline。

## 触发意图

- "写自己的 GMF element"
- "自定义音频处理 element"
- "实现 gain/特效 element"
- "element open/process/close"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `esp_gmf_audio_element.h`、`esp_gmf_oal_mem.h`、`esp_gmf_element.h` |
| 参考文档 | `docs/en/gmf-framework/gmf-core/gmf-core-element.rst`（Custom Element Template） |

## 分步说明

以「输入 PCM 乘以增益」element 为例，骨架可直接套用。

### 1. 定义派生类与配置结构（基类必须是首成员）

```c
#include "esp_gmf_audio_element.h"
#include "esp_gmf_oal_mem.h"

typedef struct {
    esp_gmf_audio_element_t parent;   // 必须是首成员，便于与基类互转
    float gain;
} gain_el_t;

typedef struct {
    float gain;
    int   data_size;
} gain_cfg_t;
```

### 2. 实现 open/process/close

```c
static esp_gmf_job_err_t gain_open(void *self, void *para)
{
    gain_el_t *el = (gain_el_t *)self;
    gain_cfg_t *cfg = (gain_cfg_t *)OBJ_GET_CFG(self);   // 取绑定配置
    el->gain = cfg->gain;
    return ESP_GMF_JOB_ERR_OK;
}

static esp_gmf_job_err_t gain_process(void *self, void *para)
{
    gain_el_t *el = (gain_el_t *)self;
    esp_gmf_port_handle_t in  = ESP_GMF_ELEMENT_GET_IN_PORT(self);
    esp_gmf_port_handle_t out = ESP_GMF_ELEMENT_GET_OUT_PORT(self);
    esp_gmf_payload_t *in_load = NULL, *out_load = NULL;

    esp_gmf_err_io_t ret = esp_gmf_port_acquire_in(in, &in_load, 1024, ESP_GMF_MAX_DELAY);
    if (ret < 0) return (ret == ESP_GMF_IO_ABORT) ? ESP_GMF_JOB_ERR_ABORT : ESP_GMF_JOB_ERR_FAIL;

    ret = esp_gmf_port_acquire_out(out, &out_load, in_load->valid_size, ESP_GMF_MAX_DELAY);
    if (ret < 0) {
        esp_gmf_port_release_in(in, in_load, 0);   // 错误分支必须先释放
        return (ret == ESP_GMF_IO_ABORT) ? ESP_GMF_JOB_ERR_ABORT : ESP_GMF_JOB_ERR_FAIL;
    }

    int16_t *src = (int16_t *)in_load->buf;
    int16_t *dst = (int16_t *)out_load->buf;
    int n = in_load->valid_size / sizeof(int16_t);
    for (int i = 0; i < n; i++) dst[i] = (int16_t)(src[i] * el->gain);

    out_load->valid_size = in_load->valid_size;
    out_load->is_done    = in_load->is_done;        // 必须传播 is_done

    esp_gmf_port_release_out(out, out_load, 0);
    esp_gmf_port_release_in(in, in_load, 0);
    return in_load->is_done ? ESP_GMF_JOB_ERR_DONE : ESP_GMF_JOB_ERR_OK;
}

static esp_gmf_job_err_t gain_close(void *self, void *para)
{
    return ESP_GMF_JOB_ERR_OK;   // 释放 open 阶段分配的资源
}
```

### 3. 构造函数：设端口属性、绑定配置、装 ops

```c
esp_gmf_err_t gain_el_init(gain_cfg_t *cfg, esp_gmf_element_handle_t *out)
{
    gain_el_t *el = esp_gmf_oal_calloc(1, sizeof(gain_el_t));
    if (!el) return ESP_GMF_ERR_MEMORY_LACK;

    esp_gmf_element_cfg_t el_cfg = { 0 };
    ESP_GMF_ELEMENT_IN_PORT_ATTR_SET(el_cfg.in_attr,
        ESP_GMF_EL_PORT_CAP_SINGLE, 16, 16, ESP_GMF_PORT_TYPE_BYTE, cfg->data_size);
    ESP_GMF_ELEMENT_OUT_PORT_ATTR_SET(el_cfg.out_attr,
        ESP_GMF_EL_PORT_CAP_SINGLE, 16, 16, ESP_GMF_PORT_TYPE_BYTE, cfg->data_size);
    el_cfg.dependency = true;   // 需上游格式信息才能 open

    esp_gmf_audio_el_init((esp_gmf_audio_element_handle_t)el, &el_cfg);
    esp_gmf_obj_set_tag((esp_gmf_obj_handle_t)el, "gain");
    esp_gmf_obj_set_config((esp_gmf_obj_handle_t)el, cfg, sizeof(gain_cfg_t));

    esp_gmf_element_t *base = ESP_GMF_ELEMENT_GET(el);
    base->ops.open    = gain_open;
    base->ops.process = gain_process;
    base->ops.close   = gain_close;

    *out = el;
    return ESP_GMF_ERR_OK;
}
```

### 4. 注册进 pool，参与 pipeline

```c
esp_gmf_element_handle_t gain_handle = NULL;
gain_cfg_t gcfg = { .gain = 1.5f, .data_size = 768 };
gain_el_init(&gcfg, &gain_handle);
esp_gmf_pool_register_element(pool, gain_handle, NULL);

// 建流水线时把 "gain" 加进 element 名数组
const char *name[] = {"aud_dec", "aud_rate_cvt", "gain"};
esp_gmf_pool_new_pipeline(pool, "io_file", name, 3, "io_codec_dev", &pipe);
```

> `dependency = true` 表示该 element 需上游 REPORT_INFO 后才注册 open/process job（如算法依赖采样率）。处理逻辑不依赖上游格式时设 false。

### 5. job 返回码约定（自定义 element 必须遵守）

| 返回码 | 含义 |
|---|---|
| `ESP_GMF_JOB_ERR_OK`(0) | 正常完成；常驻 job 保留下一轮 |
| `ESP_GMF_JOB_ERR_CONTINUE`(1) | 立即跳回首 element，本轮跳过后续（数据不足时用） |
| `ESP_GMF_JOB_ERR_DONE`(2) | 永久完成，移出 job 列表（传播上游 is_done） |
| `ESP_GMF_JOB_ERR_TRUNCATE`(3) | 当前 job 入栈，下一轮从本 element 继续（输出满时用） |
| `ESP_GMF_JOB_ERR_FAIL`(-1) | 失败，进入 ERROR 流程 |
| `ESP_GMF_JOB_ERR_ABORT`(-3) | 主动中止，按策略进 STOPPED 或 RESET |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| element 迟迟不 open | `dependency=true` 但上游没 notify | 上游 open/process 里调 `esp_gmf_element_notify_snd_info` |
| port 泄漏耗尽缓冲 | acquire/release 不成对 | 每个错误分支 release 已 acquire 的 payload |
| pipeline 不结束 | 没传播 `is_done` | out_load->is_done = in_load->is_done，返回 DONE |
| 派生类强转失败 | 基类不是首成员 | 把 `esp_gmf_audio_element_t parent` 放结构体第一字段 |

## 参考

- `docs/en/gmf-framework/gmf-core/gmf-core-element.rst`（Custom Element Template 完整代码）
- `docs/en/gmf-framework/gmf-core/gmf-core-data-path.rst`（acquire-release 协议）
- `gmf_core/test_apps/main/common/gmf_general_el.h`（测试用 element 参考）
