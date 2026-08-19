# 运行时自定义命令词

> **适用摘要**: 不依赖 menuconfig 默认命令表，在运行时用 `esp_mn_commands_*` API 动态增删改命令词，并刷新 MultiNet 语言模型。适用于 mn6/mn7（推荐）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-skainet/resources/`, source/examples in `repos/esp-skainet/`, and this recipe path `repos/esp-skainet/recipes/customize_commands.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "动态添加命令词"
- "运行时改命令词"
- "esp_mn_commands_add"
- "custom speech commands"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/cn_speech_commands_recognition/`、`examples/en_speech_commands_recognition/` |
| 依赖组件 | `espressif/esp-sr` (^2.0.0) |
| 前置步骤 | 已 `multinet->create(mn_name, 6000)` 拿到 `model_data` |
| 头文件 | `esp_mn_iface.h`（`esp_mn_error_t` 来自该头/`esp_mn_models.h`） |

## 分步说明

### 1. 命令 API 全景（来自 `esp_mn_speech_commands.h`）

```c
// 注意：以下函数操作的是全局命令链表，不绑定具体 model_data
esp_err_t esp_mn_commands_alloc(const esp_mn_iface_t *multinet, model_iface_data_t *model_data);
esp_err_t esp_mn_commands_free(void);

esp_err_t esp_mn_commands_add(int command_id, const char *string);
esp_err_t esp_mn_commands_phoneme_add(int command_id, const char *string, const char *phonemes);
esp_err_t esp_mn_commands_modify(const char *old_string, const char *new_string);
esp_err_t esp_mn_commands_remove(const char *string);
esp_err_t esp_mn_commands_clear(void);

char           *esp_mn_commands_get_string(int command_id);
esp_mn_phrase_t*esp_mn_commands_get_from_index(int index);
esp_mn_phrase_t*esp_mn_commands_get_from_string(const char *string);

// 关键：增删改之后必须调用，返回无法解析的短语列表（NULL=全部成功）
esp_mn_error_t *esp_mn_commands_update(void);
```

### 2. 标准流程：clear → add → update

```c
// 必须在 multinet->create() 之后调用
esp_mn_commands_clear();                       // 清空已有命令
esp_mn_commands_add(1, "turn on the light");   // 英文
esp_mn_commands_add(2, "turn off the light");
esp_mn_commands_add(3, "da kai kong tiao");    // 中文拼音（mn2/mn5）

esp_mn_error_t *err = esp_mn_commands_update();
if (err != NULL) {
    printf("%d phrase(s) failed to parse:\n", err->num);
    for (int i = 0; i < err->num; i++) {
        printf("  [%d] %s\n", err->phrases[i]->command_id, err->phrases[i]->string);
    }
}

multinet->print_active_speech_commands(model_data);
```

> `command_id` 是用户自定义的整数 ID，识别命中后从 `mn_result->command_id[i]` 取回，用于触发不同动作。

### 3. 增量修改 / 删除单条

```c
// 把 "turn on the light" 改成 "switch on the light"
esp_mn_commands_modify("turn on the light", "switch on the light");
esp_mn_commands_update();

// 删除一条
esp_mn_commands_remove("turn off the light");
esp_mn_commands_update();
```

### 4. 命中后取结果与 ID 映射

```c
if (mn_state == ESP_MN_STATE_DETECTED) {
    esp_mn_results_t *r = multinet->get_results(model_data);
    for (int i = 0; i < r->num; i++) {
        int id = r->command_id[i];
        printf("hit command_id=%d, phrase_id=%d, prob=%f, string=%s\n",
               id, r->phrase_id[i], r->prob[i], r->string);
        // 用 id 分发动作
        switch (id) {
            case 1: turn_on_light();  break;
            case 2: turn_off_light(); break;
        }
    }
}
```

### 5. 限制（来自头文件宏）

- `ESP_MN_MAX_PHRASE_NUM = 400` —— 命令数上限（README 标称最多 200 条中/英命令）
- `ESP_MN_MAX_PHRASE_LEN = 63` —— 单条命令字符串最大长度
- `ESP_MN_MIN_PHRASE_LEN = 2` —— 最小长度
- `ESP_MN_RESULT_MAX_NUM = 5` —— `get_results` 一次最多返回 5 个候选

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 识别不到任何命令 | add 后没调 `esp_mn_commands_update` | 务必在 add/remove/modify 后 update |
| `esp_mn_commands_update` 返回错误短语 | 命令字符串音节不合法 | 检查返回的 `esp_mn_error_t`，修正字符串 |
| 命中 command_id 全是 0 | 没用 add 设置 ID，或用了 sdkconfig 导入 | 显式 `esp_mn_commands_add(id, str)` |
| 命令字符串超长被截断 | 超过 `ESP_MN_MAX_PHRASE_LEN`(63) | 拆短或换说法 |
| 中文命令不识别 | 用了汉字而非拼音 | mn2/mn5/mn7 中文命令用拼音（如 `da kai kong tiao`） |
| 重复 add 同 ID | 链表里残留旧命令 | 先 `esp_mn_commands_clear()` 再 add |

## 参考

- `espressif-repos/esp-sr/src/include/esp_mn_speech_commands.h` — 全部命令 API 真实声明
- `espressif-repos/esp-sr/include/esp32s3/esp_mn_iface.h` — `ESP_MN_MAX_PHRASE_NUM` 等宏、`esp_mn_error_t`
- `examples/cn_speech_commands_recognition/README.md`、`examples/en_speech_commands_recognition/README.md` — add/update 示例
- `espressif-repos/esp-sr/src/include/esp_process_sdkconfig.h` — `esp_mn_commands_update_from_sdkconfig`
