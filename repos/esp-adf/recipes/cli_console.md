# CLI 命令行交互

> **适用摘要**: 用 ESP-IDF 的 `esp_console`（线性命令行）在 ESP-ADF 工程里注册命令，配合串口做交互调试与控制（如手动切歌、调音量、查状态）。ESP-ADF 自带 `cli` 示例演示命令注册与执行。

> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "命令行控制"
- "串口命令 / console"
- "CLI 注册命令"
- "esp_console"
- "idf.py monitor 里输入命令"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/cli/main/console_example.c` |
| 组件 | `esp_console`（ESP-IDF 自带）、`esp_vfs_dev`、`linenoise`（行编辑） |
| 默认配置 | `examples/cli/sdkconfig.defaults*`（按目标芯片选） |

## 分步说明

### 1. 初始化 console（VFS + linenoise）

```c
#include "esp_console.h"
#include "esp_vfs_dev.h"
#include "driver/uart.h"
#include "linenoise.h"

esp_console_config_t console_cfg = ESP_CONSOLE_CONFIG_DEFAULT();
esp_console_init(&console_cfg);

// 注册 readline / 历史等（见 esp_console 例程）
esp_console_register_help_command();
```

### 2. 注册自定义命令（标准模式）

```c
static int cmd_play(int argc, char **argv)
{
    // 解析 argv，触发 pipeline run / 切歌
    return 0;
}

static void register_play(void)
{
    const esp_console_cmd_t cmd = {
        .command = "play",
        .help    = "Start / resume playback",
        .func    = cmd_play,
    };
    ESP_ERROR_CHECK(esp_console_cmd_register(&cmd));
}
```

### 3. 命令分发主循环

```c
while (1) {
    char *line = linenoise("adf> ");
    if (line == NULL) continue;
    int ret;
    esp_err_t err = esp_console_run(line, &ret);
    if (err != ESP_OK) {
        printf("err: %s\n", esp_err_to_name(err));
    }
    linenoiseFree(line);
}
```

> `examples/cli/` 还演示了 BLE GATT 触发命令（`ble_gatts_module.c`），即用手机 BLE 下发命令到同一个命令处理器。

### 4. 与 audio pipeline 联动

命令回调里调用 pipeline 控制接口（注意线程安全：console 任务与 element 任务不同，跨线程操作 element 用其提供的 API，不要直接读写其内部缓冲）：
```c
// cmd_play 内：
audio_pipeline_resume(pipeline);   // 或 run / pause / stop
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 命令不显示 | VFS 未挂到 UART | 初始化 `esp_vfs_dev_*` 使用 UART0 |
| 输入乱码 | monitor 波特与 UART 配置不符 | 默认 115200；`idf.py monitor` 自动适配 |
| 回调里崩 | 跨线程直接改 element 内部 | 用 `audio_pipeline_*` / `audio_element_*` 公开 API |
| 命令注册失败 | `esp_console_cmd_t` 字段不全 | command/func 必填 |

## 参考项目

- `examples/cli/main/console_example.c` — console 命令注册与分发
- `examples/cli/main/ble_gatts_module.c` — BLE 触发命令
- `examples/cli/sdkconfig.defaults*` — 各芯片默认配置
