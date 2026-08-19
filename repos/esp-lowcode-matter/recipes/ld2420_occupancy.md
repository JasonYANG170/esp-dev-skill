# LD2420 雷达占用传感器（UART）

> **适用摘要**: 用 `occupancy_sensor_ld2420_init` 初始化 LD2420，进入 normal/report 模式，周期读取占用状态与距离，以 `LOW_CODE_FEATURE_ID_OCCUPANCY_SENSOR_VALUE` 上报。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-lowcode-matter/resources/`, source/examples in `repos/esp-lowcode-matter/`, and this recipe path `repos/esp-lowcode-matter/recipes/ld2420_occupancy.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "人体存在检测"
- "LD2420"
- "雷达占用传感器"
- "occupancy sensor lowcode"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `occupancy_sensor_ld2420.h`（带入 `uart.h`） |
| 组件依赖 | REQUIRES 含 `occupancy_sensor_ld2420` |
| 参考产品 | `products/occupancy_sensor` |

## 分步说明

### 1. 配置与数据结构（`occupancy_sensor_ld2420.h`）

```c
typedef void* occupancy_sensor_ld2420_handle_t;

typedef struct {
    uart_port_t uart_num;
    int tx_pin;
    int rx_pin;
    int ot_pin;          /* 可选控制脚 */
} occupancy_sensor_ld2420_cfg_t;

typedef struct {
    uint8_t  occupied;   /* 1=有人, 0=无人 */
    uint16_t range;      /* 距离 cm */
} occupancy_sensor_ld2420_normal_mode_data_t;

typedef struct {
    uint8_t  occupied;
    uint16_t target_distance;
    uint16_t zone_noise_level[16];   /* 16 个门区噪声 */
} occupancy_sensor_ld2420_report_mode_data_t;
```

### 2. API（节选）

```c
occupancy_sensor_ld2420_handle_t occupancy_sensor_ld2420_init(occupancy_sensor_ld2420_cfg_t *cfg);
int occupancy_sensor_ld2420_get_firmware_version(handle, char *buffer, size_t size);
int occupancy_sensor_ld2420_set_minimum_distance(handle, uint16_t minimum_distance);
int occupancy_sensor_ld2420_set_maximum_distance(handle, uint16_t maximum_distance);
int occupancy_sensor_ld2420_set_absence_report_delay(handle, uint16_t delay_s);
int occupancy_sensor_ld2420_set_gate_trigger_threshold(handle, uint8_t gate_index, uint16_t threshold);
int occupancy_sensor_ld2420_set_gate_hold_threshold(handle, uint8_t gate_index, uint16_t threshold);
int occupancy_sensor_ld2420_enter_normal_mode(handle);
int occupancy_sensor_ld2420_read_normal_data(handle, occupancy_sensor_ld2420_normal_mode_data_t *data);
int occupancy_sensor_ld2420_enter_report_mode(handle);
int occupancy_sensor_ld2420_read_report_data(handle, occupancy_sensor_ld2420_report_mode_data_t *data);
```

### 3. 完整初始化（取自 `products/occupancy_sensor/main/app_driver.cpp`）

使用 HP UART PORT 1，TX=GPIO3，RX=GPIO2：

```cpp
#include <occupancy_sensor_ld2420.h>

#define LD2420_UART_PORT_NUM UART_NUM_1
#define LD2420_RX_GPIO_NUM   (gpio_num_t)GPIO_NUM_2
#define LD2420_TX_GPIO_NUM   (gpio_num_t)GPIO_NUM_3

static occupancy_sensor_ld2420_handle_t handle;

/* 在 app_driver_init 中 */
occupancy_sensor_ld2420_cfg_t ld2420_cfg = {
    .uart_num = LD2420_UART_PORT_NUM,
    .tx_pin   = LD2420_TX_GPIO_NUM,
    .rx_pin   = LD2420_RX_GPIO_NUM,
};
handle = occupancy_sensor_ld2420_init(&ld2420_cfg);
if (!handle) { printf("%s: LD2420 init failed\n", TAG); return -1; }

/* 读固件版本，确认通信正常 */
char fw[10];
if (occupancy_sensor_ld2420_get_firmware_version(handle, fw, sizeof(fw))) {
    printf("%s: LD2420 failed to get firmware version\n", TAG);
    return -1;
}
printf("%s: LD2420 firmware version: %s\n", TAG, fw);

/* 进入 normal 模式 */
if (occupancy_sensor_ld2420_enter_normal_mode(handle)) {
    printf("%s: LD2420 failed to enter normal mode\n", TAG);
    return -1;
}

/* 每 2 秒周期读取并上报 */
system_timer_handle_t timer = system_timer_create(app_driver_read_and_report_feature, NULL, 2000, true);
system_timer_start(timer);
```

### 4. 周期读取并上报

```cpp
void app_driver_report_occupancy_sensor_state(bool occupancy_state)
{
    low_code_feature_data_t feature = {
        .details = { .endpoint_id = 1, .feature_id = LOW_CODE_FEATURE_ID_OCCUPANCY_SENSOR_VALUE },
        .value = {
            .type = LOW_CODE_VALUE_TYPE_UNSIGNED_INTEGER,   /* 0/1 */
            .value_len = sizeof(uint8_t),
            .value = (uint8_t*)&occupancy_state
        },
    };
    low_code_feature_update_to_system(&feature);
}

void app_driver_read_and_report_feature(system_timer_handle_t timer_handle, void *user_data)
{
    occupancy_sensor_ld2420_normal_mode_data_t data;
    if (occupancy_sensor_ld2420_read_normal_data(handle, &data)) {
        printf("%s: LD2420 unable to fetch data\n", TAG);
        return;
    }
    app_driver_report_occupancy_sensor_state(data.occupied);
    /* 如需距离：data.range */
}
```

### 5. report 模式（详细 16 门区噪声）

```cpp
occupancy_sensor_ld2420_enter_report_mode(handle);
occupancy_sensor_ld2420_report_mode_data_t rdata;
occupancy_sensor_ld2420_read_report_data(handle, &rdata);
/* rdata.target_distance, rdata.zone_noise_level[0..15] */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| init 返回 NULL | UART 引脚/端口错 | 确认 TX/RX 与硬件一致（注意交叉） |
| 固件版本读不到 | 通信未建立 | 先 init 再读 fw 校验；检查电平/波特 |
| 检测不准 | 门限未调 | 用 `set_minimum/maximum_distance` 与 `set_gate_*_threshold` 调参 |
| 误报不断 | 离开延时短 | `set_absence_report_delay` 加大 |
| 上报值类型错 | 用了 BOOLEAN | 示例用 `UNSIGNED_INTEGER` (uint8_t) |

## 参考

- `components/occupancy_sensor_ld2420/occupancy_sensor_ld2420.h`、`components/occupancy_sensor_ld2420/Kconfig`
- `products/occupancy_sensor/main/app_driver.cpp`
- `recipes/system_timer.md`
