# 示例索引（examples/）

> 全部路径取自仓库 `examples/`。每个示例均可作为项目模板复制后修改。

## Home Automation Devices（ZHA 标准设备）

| 路径 | 说明 | 角色 |
|---|---|---|
| `examples/home_automation_devices/on_off_light/` | HA on/off 灯，ZC 形成网络 + 开放入网，ZCL SET_ATTR_VALUE 驱动 LED | ZC |
| `examples/home_automation_devices/on_off_switch/` | HA on/off 开关，ZED 入网 + ZDO match/bind 发现绑定灯，按键 toggle | ZED |
| `examples/home_automation_devices/color_dimmable_light/` | 彩光灯（on/off + level + color_control），ZC | ZC |
| `examples/home_automation_devices/color_dimmer_switch/` | 彩灯开关（ZR），发 level/color 命令 | ZR |
| `examples/home_automation_devices/temperature_sensor/` | 温度传感器，set_attr_value 更新读数 + report_attr_cmd 上报 | ZR/ZED |
| `examples/home_automation_devices/thermostat/` | 温控器（thermostat cluster），ZC | ZC |

## Sleepy Devices（休眠终端）

| 路径 | 说明 |
|---|---|
| `examples/sleepy_devices/light_sleep_end_device/` | Light sleep 终端 switch：RxOnWhenIdle=false、EXT1 唤醒、`esp_pm` |
| `examples/sleepy_devices/deep_sleep_end_device/` | Deep sleep 终端：`esp_deep_sleep_start()` 整芯片断电、RTC timer + EXT1(BOOT) 唤醒、wake-from-reset、唤醒后协议栈 rejoin；`RTC_DATA_ATTR` 跨睡眠保留时间戳。sdkconfig 关键项 `CONFIG_BOOTLOADER_SKIP_VALIDATE_IN_DEEP_SLEEP=y`、`CONFIG_NEWLIB_TIME_SYSCALL_USE_RTC_HRT=y`、`CONFIG_RTC_CLK_SRC_INT_RC=y` |

## Touchlink

| 路径 | 说明 |
|---|---|
| `examples/touchlink/touchlink_initiator/` | Touchlink 发起方，扫描并拉起 target 入网 |
| `examples/touchlink/touchlink_target/` | Touchlink 目标方，等待被发起 |

## OTA Upgrade

| 路径 | 说明 |
|---|---|
| `examples/ota_upgrade/ota_server/` | OTA 升级服务端（下发镜像） |
| `examples/ota_upgrade/ota_client/` | OTA 升级客户端（接收 + 写分区 + 切换启动），支持 delta OTA |

## Zigbee Gateway（网关 / RCP）

| 路径 | 说明 |
|---|---|
| `examples/zigbee_gateway/` | 主控 + RCP（UART）网关，可选 Wi-Fi/Ethernet 回程、共存、RCP 自动升级；含 `generate_rcp_image.py`、`sdkconfig.defaults.esp32s3`/`esp32p4` |

## Customized Devices（自定义 cluster）

| 路径 | 说明 |
|---|---|
| `examples/customized_devices/data_producer/` | 自定义 data stream cluster 服务端（生产数据，响应查询） |
| `examples/customized_devices/data_consumer/` | 自定义 data stream cluster 客户端（消费数据，发起查询） |

## All-in-One

| 路径 | 说明 |
|---|---|
| `examples/all_device_types_app/` | 单节点多 endpoint 演示多种 ZHA 设备类型 |

## Utils（公共组件，被多个示例复用）

| 路径 | 说明 |
|---|---|
| `examples/utils/alarm_timer/` | 延迟重试定时器（commissioning 失败重试用） |
| `examples/utils/light_driver/` | LED 驱动（WS2812/单色）封装 |
| `examples/utils/switch_driver/` | 按键驱动，事件回调 |
| `examples/utils/temp_sensor_driver/` | 片内温度传感器驱动（基于 ESP-IDF temperature_sensor） |
| `examples/utils/example_common/` | 示例公共代码 |
| `examples/utils/delta_ota/` | Delta OTA 工具（配合 `CONFIG_ZB_DELTA_OTA`） |

## 测试脚本（仓库根）

| 文件 | 说明 |
|---|---|
| `examples/README.md` | 示例总览与运行说明 |
| `examples/zigbee_common.py` | pytest 公共工具 |
| `examples/constants.py` | 测试常量 |
| `examples/pytest_esp_zigbee_open_examples.py` | 开放示例自动化测试 |
| `examples/pytest_esp_zigbee_cli.py` | CLI 示例测试 |
